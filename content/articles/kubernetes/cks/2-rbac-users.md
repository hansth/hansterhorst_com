---
title: RBAC Users
date: 2026-10-22
weight: 2
tags:
  - CKS
  - Security
  - Kubernetes
  - kubeadm
  - RBAC
  - Certificates
  - Tutorial
---
# Project RBAC Users

In this project I create two users for myself in my homelab cluster:

- `hans-adm`: Admin for cluster maintenance
- `hans-dev`: Normal user for daily work in the `dev` Namespace.

Until now, I used the `kubernetes-admin` credentials from `/etc/kubernetes/admin.conf` for everything. With my own users, every request with the user name will be logged in the audit log, and the normal user can't break the cluster.

Each user gets a client certificate signed by the cluster Certificate Authority (CA), permissions through RBAC, and a kubeconfig file. I run all commands on the first control plane as `kubernetes-admin`, unless a step says otherwise.

---

## The Plan

Both users get their permissions from a group, not from their user name. The group is part of the certificate, and the RBAC grants the permissions to that group.

| User       | Group            | ClusterRole     | RBAC                                | Scope             | Certificate valid |
| ---------- | ---------------- | --------------- | ----------------------------------- | ----------------- | ----------------- |
| `hans-adm` | `homelab-admins` | `cluster-admin` | ClusterRoleBinding `homelab-admins` | The whole cluster | 30 days           |
| `hans-dev` | `homelab-devs`   | `edit`          | RoleBinding `homelab-devs`          | Namespace `dev`   | 90 days           |
- `cluster-admin`: A ClusterRole with full access to everything.
- `edit`: A ClusterRole that can create and change most objects in a Namespace, but can't change Roles, RoleBindings, or the Namespace itself.

The ClusterRoles are maintained by Kubernetes and are updated when new resources are added.

### The Tasks

The project takes six steps. For practice, I work through the sub-tasks with only the Kubernetes documentation open, as in the exam, and afterwards check my work against the steps below. Each step lists the docs pages and search terms for [kubernetes.io/docs](https://kubernetes.io/docs/).

1. [Create the certificates](#step-1-create-the-certificates)
    - Create a 4096-bit RSA private key for each user.
    - Create a CSR with the user name as `CN` and the group as `O`.
    - Create and apply the CSR as a CertificateSigningRequest with the `kubernetes.io/kube-apiserver-client` signer, `client auth` usage, and a lifetime of 30 days for `hans-adm` and 90 days for `hans-dev`.
    - Approve the request and save the signed certificate to a file.
    - Check the subject, issuer, and expiry date with `openssl`.
    - Docs: [Issue a Certificate for a Kubernetes API Client Using A CertificateSigningRequest](https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/), [Generate Certificates Manually](https://kubernetes.io/docs/tasks/administer-cluster/certificates/). Search: `certificate signing request`, `manage tls`, `manual openssl`.
2. [Grant the permissions](#step-2-grant-the-permissions)
    - Bind the built-in `cluster-admin` ClusterRole to the group `homelab-admins` for the whole cluster.
    - Create the Namespace `dev`.
    - Bind the built-in `edit` ClusterRole to the group `homelab-devs` in `dev` Namespace only.
    - Check with `kubectl auth can-i` that `hans-dev` can create Deployments in `dev`, but not in `default` Namespace, and can't create RoleBindings.
    - Docs: [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), section User-facing roles. Search: `rbac user-facing roles`.
3. [Build the kubeconfig files](#step-3-build-the-kubeconfig-files)
    - Create a kubeconfig file per user, without copying `admin.conf`.
    - Add the cluster with the HAProxy endpoint and the embedded cluster CA.
    - Add the user with the embedded certificate and key.
    - Create a context and make it the current context. Set `dev` as the default Namespace for `hans-dev`.
    - Restrict the file permissions to the owner.
    - Docs: [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/). Search: `configure access multiple clusters`.
4. [Test the users](#step-4-test-the-users)
    - Check the user name and groups of each kubeconfig.
    - As `hans-adm`, list the nodes.
    - As `hans-dev`, create a Deployment in `dev`, and confirm that Pods in `default` are forbidden and that listing nodes is forbidden.
    - Docs: [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/), sections X.509 client certificates and API access to authentication information for a client. Search: `kubectl auth whoami`.
5. [Move the kubeconfig files to the laptop](#step-5-move-the-kubeconfig-files-to-the-laptop)
    - Copy both kubeconfig files to `~/.kube` on the laptop, and restrict their permissions.
    - Use `hans-dev` by default, and `hans-adm` only for single commands.
    - Remove the keys, CSRs, and kubeconfig files from the node.
    - Docs: [Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/). Search: `kubeconfig KUBECONFIG environment variable`.
6. [Check for risky bindings](#step-6-check-for-risky-bindings)
    - List every ClusterRoleBinding to `cluster-admin`, and explain each one.
    - List the RoleBindings in `dev`.
    - Confirm that `hans-dev` can't create Pods in `kube-system`.
    - Docs: [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/). Search: `rbac good practices`.

After the setup, [Renewing a Certificate](#renewing-a-certificate) and [Removing a User](#removing-a-user) cover the maintenance.

> **_Warning:_** Never put a user in the `system:masters` group. Members of that group bypass RBAC completely, and you can't take that access away without replacing the cluster CA.

---

## Prerequisites

| Item                 | Value                                    |
| -------------------- | ---------------------------------------- |
| Cluster              | kubeadm, 3 control planes, 1 worker node |
| Control Plane        | `k8s-master-1`                           |
| HAProxy API endpoint | `https://192.168.4.110:6443`             |
| Cluster CA           | `/etc/kubernetes/pki/ca.crt`             |
| Admin access         | `kubernetes-admin`                       |
| Tools                | `openssl`, `kubectl`                     |

---

## Step 1: Create the Certificates

Each user needs a private key and a certificate signed by the cluster CA. Kubernetes reads the user name from the `CN` and the group from the `O` of the certificate. The Certificates API signs the request, without access to the CA key.

### Prepare a Working Directory

The private keys must not be readable by other users on the node.

**Create the directory**
```bash
mkdir -p ~/rbac-users && \
chown $USER:$USER ~/rbac-users && \
chmod 700 ~/rbac-users && \
cd ~/rbac-users
```

### Set the User

The steps below use variables. I run them once for `hans-adm`, and then again for `hans-dev`.

**Set the variables for the admin user**
```bash
K8S_USER=hans-adm; K8S_GROUP=homelab-admins; K8S_DAYS=30
echo "$K8S_USER, $K8S_GROUP, $K8S_DAYS"
#hans-adm, homelab-admins, 30
```
- `K8S_USER`: The user name, which becomes the `CN` of the certificate.
- `K8S_GROUP`: The group, which becomes the `O` of the certificate.
- `K8S_DAYS`: How long the certificate is valid.

> **_Note:_** Don't use `USER` as the variable name. The shell sets `USER` to the name of the logged-in Linux user.

### Create the Key and the CSR

`openssl` creates the private key and a Certificate Signing Request (CSR) with the user name and group.

**Create the private key**
```bash
openssl genrsa -out $K8S_USER.key 4096
chmod 600 $K8S_USER.key
```
This generates an RSA private key in 4096-bit format, which is highly secure.

**Create the certificate signing request**
```bash
openssl req -new -key $K8S_USER.key -out $K8S_USER.csr \
  -subj "/CN=$K8S_USER/O=$K8S_GROUP"
```
- `openssl req`: The certificate request utility in OpenSSL.
- `-new`: Generate a new certificate request.
- `-key`: Use this existing private key.
- `-out`: Output the Certificate Signing Request (CSR) to this file.
- `-subj`: Set certificate subject fields:
    - `/CN=`: Common Name (user).
    - `/O=`: Organization (group).

In Kubernetes RBAC, the `CN` becomes the **user name**, and the `O` becomes the **group**. Add more `/O=` parts to put the user in more groups.

**Check the subject**
```bash
openssl req -in $K8S_USER.csr -noout -subject
# subject=CN = hans-adm, O = homelab-admins
```

### Submit the CSR to Kubernetes

The CSR goes into a CertificateSigningRequest object, base64-encoded. The manifest follows the template from the [Kubernetes documentation](https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/#create-k8s-certificatessigningrequest).

**Create the CertificateSigningRequest**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: $K8S_USER
spec:
  request: $(cat $K8S_USER.csr | base64 | tr -d "\n")
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: $(($K8S_DAYS * 86400))
  usages:
  - client auth
EOF
```
- `request`: The CSR as one line of base64.
- `signerName: kubernetes.io/kube-apiserver-client`: The signer for client certificates that authenticate to the kube-apiserver.
- `expirationSeconds`: The requested lifetime. The controller manager caps it at its `--cluster-signing-duration`, which is 1 year by default.
- `usages: [client auth]`: The certificate may only be used to log in as a client.

**Check the request**
```bash
kubectl get csr $K8S_USER
```
```
NAME      AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
hans-adm  5s    kubernetes.io/kube-apiserver-client   kubernetes-admin   30d                 Pending
```
The CSR shows `Pending` until an administrator approves it.

### Approve and Extract the Certificate

The cluster admin approves the request and saves the signed certificate. A pending request remains for 24 hours, while an approved one remains for only one hour.

**Approve the request**
```bash
kubectl certificate approve $K8S_USER
# certificatesigningrequest.certificates.k8s.io/hans-adm approved
```

> **_Note:_** Kubernetes deletes approved CSR objects after an hour. Save the certificate right after approval.

**Save the signed certificate**
```bash
kubectl get csr $K8S_USER -o jsonpath='{.status.certificate}' | base64 -d > $K8S_USER.crt
```

**Check the subject, issuer, and expiry date**
```bash
openssl x509 -in $K8S_USER.crt -noout -subject -issuer -enddate
```
```
subject=O = homelab-admins, CN = hans-adm
issuer=CN = kubernetes
notAfter=Nov 25 09:20:20 2026 GMT
```

### Repeat for the Normal User

Set the variables for `hans-dev`, and run the steps from [Create the Key and the CSR](#create-the-key-and-the-csr) to [Approve and Extract the Certificate](#approve-and-extract-the-certificate) again.

**Set the variables for the normal user**
```bash
K8S_USER=hans-dev
K8S_GROUP=homelab-devs
K8S_DAYS=90
```

**Change the permissions**
```bash
chmod 600 hans-*
ls -l ~/rbac-users
```
```
-rw------- 1 master-1 master-1 1472 Oct 26 10:28 hans-adm.crt
-rw------- 1 master-1 master-1 1618 Oct 26 10:19 hans-adm.csr
-rw------- 1 master-1 master-1 3268 Oct 26 10:18 hans-adm.key
-rw------- 1 master-1 master-1 1468 Oct 26 10:31 hans-dev.crt
-rw------- 1 master-1 master-1 1614 Oct 26 10:30 hans-dev.csr
-rw------- 1 master-1 master-1 3272 Oct 26 10:30 hans-dev.key
```

---

## Step 2: Grant the Permissions

A certificate only proves who a user is. Without a ClusterRoleBinding or RoleBinding, both users are authenticated but can't do anything.

### Built-in Roles or a Custom Role

RBAC always needs two objects: a role that lists what is allowed, and a binding that grants the role to users or groups. I don't create a role myself. Kubernetes ships [user-facing ClusterRoles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) and both users get one of them.

| ClusterRole     | Usually bound with | Access                                                                   |
| --------------- | ------------------ | ------------------------------------------------------------------------ |
| `cluster-admin` | ClusterRoleBinding | Everything in the cluster                                                |
| `admin`         | RoleBinding        | Everything in a Namespace, including Roles and RoleBindings              |
| `edit`          | RoleBinding        | Most objects in a Namespace, including Secrets, but no Roles or bindings |
| `view`          | RoleBinding        | Read-only access in a Namespace, without Secrets                         |

Kubernetes maintains these ClusterRoles and adds new resources to them during upgrades. If a built-in role grants more access than needed, I create a Role with just the resources and verbs the user needs.

### The Admin Group

The admin needs full access for cluster maintenance. The built-in `cluster-admin` ClusterRole already grants that, and a custom copy would only add work.

**Show the cluster-admin ClusterRole**
```bash
kubectl describe clusterrole cluster-admin
```
```
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```
- `*.*` with `[*]`: Every verb on every resource in every API group.

A ClusterRoleBinding grants `cluster-admin` to the `homelab-admins` group in the whole cluster.

**Bind cluster-admin to the admin group**
```bash
kubectl create clusterrolebinding homelab-admins \
  --clusterrole=cluster-admin \
  --group=homelab-admins
```

### The Developer Group

The developer runs workloads in `dev` Namespace only. The built-in `edit` ClusterRole can create and change most objects in a Namespace, but can't change Roles, RoleBindings, or the Namespace itself.

**Create the Namespace**
```bash
kubectl create namespace dev
```

**Show the rules of the edit ClusterRole**
```bash
kubectl describe clusterrole edit
```
```
Name:         edit
Labels:       kubernetes.io/bootstrapping=rbac-defaults
              rbac.authorization.k8s.io/aggregate-to-admin=true
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources          Non-Resource URLs  Resource Names  Verbs
  ---------          -----------------  --------------  -----
  ...
  pods               []                 []              [create delete deletecollection patch update get list watch]
  ...
  deployments.apps   []                 []              [create delete deletecollection patch update get list watch]
  ...
  pods/exec          []                 []              [get list watch create delete deletecollection patch update]
  ...
  secrets            []                 []              [get list watch create delete deletecollection patch update]
  ...
  serviceaccounts    []                 []              [impersonate create delete deletecollection patch update get list watch]
```
The output lists every resource with its verbs. It covers Pods, Deployments, Services, ConfigMaps, and Secrets, but no Roles or RoleBindings.

A RoleBinding grants `edit` to the `homelab-devs` group in the `dev` Namespace only. A RoleBinding can reference a ClusterRole: the permissions then apply only in the Namespace of the RoleBinding. This is how one ClusterRole serves every Namespace, without a copy of the Role in each one.

**Bind edit to the developer group in the dev Namespace**
```bash
kubectl -n dev create rolebinding homelab-devs \
  --clusterrole=edit \
  --group=homelab-devs
```

### Check the Permissions Before Testing

`kubectl auth can-i` with `--as` and `--as-group` checks the permissions without the certificates.

**Check the admin**
```bash
kubectl auth can-i '*' '*' --as=hans-adm --as-group=homelab-admins
# yes
```

**Check the normal user in the dev Namespace**
```bash
kubectl auth can-i create deployments -n dev --as=hans-dev \
  --as-group=homelab-devs
# yes
```

**Check the normal user outside the dev Namespace**
```bash
kubectl auth can-i create deployments -n default --as=hans-dev \
  --as-group=homelab-devs
# no
```

**Check that the normal user can't change RBAC**
```bash
kubectl auth can-i create rolebindings -n dev --as=hans-dev \
  --as-group=homelab-devs
# no
```

---

## Step 3: Build the kubeconfig Files

Each user gets a kubeconfig file of their own.

**Set the cluster with the API endpoint and the CA**
```bash
kubectl config set-cluster homelab \
  --server=https://192.168.4.110:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --kubeconfig=$K8S_USER.conf
```
- `--server`: The API endpoint of the cluster.
- `--certificate-authority`: The cluster CA, which `kubectl` uses to verify the kube-apiserver.
- `--embed-certs=true`: Copies the certificate into the file. The kubeconfig then works on any machine.

**Add the user credentials**
```bash
kubectl config set-credentials $K8S_USER \
  --client-certificate=$K8S_USER.crt \
  --client-key=$K8S_USER.key \
  --embed-certs=true \
  --kubeconfig=$K8S_USER.conf
```

**Create the context and make it the current one**
```bash
kubectl config set-context $K8S_USER@homelab \
  --cluster=homelab \
  --user=$K8S_USER \
  --kubeconfig=$K8S_USER.conf
```

**Make it the current context**
```bash
kubectl config use-context $K8S_USER@homelab --kubeconfig=$K8S_USER.conf
```

**Set the default Namespace for the normal user**
```bash
kubectl config set-context hans-dev@homelab --namespace=dev --kubeconfig=hans-dev.conf
```

> **_Note:_** Run this step only for `hans-dev`. The admin works in the whole cluster and has no default Namespace.

**Protect the file**
```bash
chmod 600 $K8S_USER.conf
```

**View the kubeconfig**
```bash
kubectl config view --kubeconfig=$K8S_USER.conf
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.4.110:6443
  name: homelab
contexts:
- context:
    cluster: homelab
    namespace: dev
    user: hans-dev
  name: hans-dev@homelab
current-context: hans-dev@homelab
kind: Config
users:
- name: hans-dev
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

> **_Warning:_** The kubeconfig contains the private key. Anyone with this file is the user.

Run the steps for both users: `K8S_USER` still holds the last user, `hans-dev`. Set `K8S_USER=hans-adm` and repeat the steps for the admin, without the default Namespace.

---

## Step 4: Test the Users

Now test this with the real certificates, not with `--as`.

### Who Am I

`kubectl auth whoami` shows the user name and the groups the kube-apiserver sees.

**Check the admin identity**
```bash
kubectl --kubeconfig=hans-adm.conf auth whoami
```
```
ATTRIBUTE   VALUE
Username    hans-adm
Groups      [homelab-admins system:authenticated]
```

**Check the normal user identity**
```bash
kubectl --kubeconfig=hans-dev.conf auth whoami
```
```
ATTRIBUTE   VALUE
Username    hans-dev
Groups      [homelab-devs system:authenticated]
```

### The Admin

The admin can read and change everything in the cluster.

**List the nodes**
```bash
kubectl --kubeconfig=hans-adm.conf get nodes
```
```
NAME           STATUS   ROLES           AGE    VERSION
k8s-master-1   Ready    control-plane   180d   v1.35.5
k8s-master-2   Ready    control-plane   180d   v1.35.5
k8s-master-3   Ready    control-plane   180d   v1.35.5
k8s-worker-1   Ready    <none>          180d   v1.35.5
```

### The Normal User

The normal user can run workloads in `dev`, and nothing else.

**Create a Deployment in dev**
```bash
kubectl --kubeconfig=hans-dev.conf create deployment nginx --image=nginx
# deployment.apps/nginx created
```

**Try to create a Pod in default**
```bash
kubectl --kubeconfig=hans-dev.conf -n default run test --image=nginx
```
```
Error from server (Forbidden): pods is forbidden: User "hans-dev" cannot create resource "pods" in API group "" in the namespace "default"
```

**Try to list the nodes**
```bash
kubectl --kubeconfig=hans-dev.conf get nodes
```
```
Error from server (Forbidden): nodes is forbidden: User "hans-dev" cannot list resource "nodes" in API group "" at the cluster scope
```

**List everything the normal user can do in dev**
```bash
kubectl auth can-i --list -n dev --as=hans-dev --as-group=homelab-devs
```
```
Resources          Non-Resource URLs   Resource Names   Verbs
...
pods               []                  []               [create delete deletecollection patch update get list watch]
...
deployments.apps   []                  []               [create delete deletecollection patch update get list watch]
...
pods/exec          []                  []               [get list watch create delete deletecollection patch update]
...
secrets            []                  []               [get list watch create delete deletecollection patch update]
...
serviceaccounts    []                  []               [impersonate create delete deletecollection patch update get list watch]
```

> **_Note:_** `edit` can read Secrets in its Namespace. If the normal user only needs to look around, bind `view` instead, which excludes Secrets.

> **_Warning:_** `edit` also allows `impersonate` on `serviceaccounts`. The normal user can act as any ServiceAccount in `dev`, with all of its permissions. Don't bind extra permissions to a ServiceAccount in a Namespace where users have `edit`.

---

## Step 5: Move the kubeconfig Files to the Laptop

The kubeconfig files and keys belong on the laptop, not on a control plane node. Anyone with root on the node could copy them.

**Copy the files from the laptop**
```bash
scp k8s-master-1:~/rbac-users/hans-adm.conf ~/.kube/hans-adm.conf
scp k8s-master-1:~/rbac-users/hans-dev.conf ~/.kube/hans-dev.conf
chmod 600 ~/.kube/hans-*.conf
```

**Use the normal user by default on the laptop**
```bash
export KUBECONFIG=~/.kube/hans-dev.conf
kubectl get pods
```

**Use the admin for a single command**
```bash
kubectl --kubeconfig ~/.kube/hans-adm.conf get nodes
```

**Remove the keys and files from the node**
```bash
rm ~/rbac-users/*.key ~/rbac-users/*.csr ~/rbac-users/*.conf
```

The `.crt` files can stay. They hold no secrets, and `openssl` shows their expiry date.

> **_Note:_** I keep `/etc/kubernetes/admin.conf` on the control plane nodes as a rescue account, for when the certificate expires or a binding breaks. Since Kubernetes v1.29, kubeadm also creates `/etc/kubernetes/super-admin.conf`, which is in `system:masters`.

---

## Step 6: Check for Risky Bindings

A CKS task may ask to identify who has excessive access. The same checks show that my setup is as intended.

**List the bindings to cluster-admin**
```bash
kubectl get clusterrolebindings -o wide | grep -E 'NAME|cluster-admin'
```
```
NAME                     ROLE                        AGE    USERS   GROUPS                    SERVICEACCOUNTS
cluster-admin            ClusterRole/cluster-admin   180d           system:masters
homelab-admins           ClusterRole/cluster-admin   10m            homelab-admins
kubeadm:cluster-admins   ClusterRole/cluster-admin   180d           kubeadm:cluster-admins
```
- `cluster-admin`: Created by Kubernetes for `system:masters`.
- `kubeadm:cluster-admins`: Created by kubeadm for `kubernetes-admin` in `admin.conf`.
- `homelab-admins`: My admin group.

Anything else bound to `cluster-admin`, especially a ServiceAccount, needs a reason.

**List the RoleBindings in dev**
```bash
kubectl -n dev get rolebindings -o wide
```
```
NAME           ROLE               AGE   USERS   GROUPS         SERVICEACCOUNTS
homelab-devs   ClusterRole/edit   43m           homelab-devs
```

**Check who may create Pods in kube-system**
```bash
kubectl auth can-i create pods -n kube-system --as=hans-dev --as-group=homelab-devs
# no
```

---

## Renewing a Certificate

A client certificate can't be revoked or extended: it is valid until it expires. Before that, I request a new one with the same key.

**Check when the certificate expires**
```bash
kubectl config view --raw --kubeconfig ~/.kube/hans-adm.conf \
  -o jsonpath='{.users[0].user.client-certificate-data}' | \
  base64 -d | openssl x509 -noout -enddate
# notAfter=Nov 25 09:20:20 2026 GMT
```

To renew, extract the key from the kubeconfig, run [Step 1](#step-1-create-the-certificates) from the CSR, and update the credentials in the kubeconfig with `kubectl config set-credentials`.

**Extract the key from the kubeconfig**
```bash
kubectl config view --raw --kubeconfig ~/.kube/hans-adm.conf \
  -o jsonpath='{.users[0].user.client-key-data}' | base64 -d > hans-adm.key
chmod 600 hans-adm.key
```

---

## Removing a User

The certificate remains valid until it expires, even when the user should lose access. The permissions come from the RBAC, not from the certificate. Removing the RBAC immediately revokes access.

**Remove the admin access**
```bash
kubectl delete clusterrolebinding homelab-admins
```

**Check that the access is gone**
```bash
kubectl auth can-i get nodes --as=hans-adm --as-group=homelab-admins
# no
```

> **_Warning:_** The group is part of the certificate. Any new binding for `homelab-admins` gives the old certificate its access back. Pick a new group name when you set up the access again.

---

## Common Errors

| Error                                                                        | Cause                                                           | Fix                                                                              |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `error: You must be logged in to the server (Unauthorized)`                  | The certificate expired, or wasn't signed by the cluster CA     | Check `notAfter` with `openssl`, and renew the certificate                       |
| `User "hans-dev" cannot list resource "pods" ... in the namespace "default"` | The RoleBinding is only in `dev` Namespace                      | Use `-n dev`, or set the Namespace in the context                                |
| `kubectl get csr hans-adm` returns `NotFound`                                | The approved CSR was cleaned up after about an hour             | Create and approve a new CSR, and save the certificate right away                |
| `x509: certificate signed by unknown authority`                              | The kubeconfig has the wrong CA, or `--embed-certs` was missing | Set the cluster again with `/etc/kubernetes/pki/ca.crt` and `--embed-certs=true` |
| `auth whoami` shows no `homelab-admins` group                                | The `O=` was missing in the CSR subject                         | Create a new CSR with `-subj "/CN=hans-adm/O=homelab-admins"`                    |
| `base64: invalid option -- 'w'` on macOS                                     | macOS `base64` has no `-w0` option                              | Run the steps on the control plane node, or use `base64 < file \| tr -d '\n'`    |

---

## Cleanup

Only for a test run. In my homelab, I keep both users.

**Delete the bindings, the Namespace, and the CSRs**
```bash
kubectl delete clusterrolebinding homelab-admins
kubectl delete namespace dev
kubectl delete csr hans-adm hans-dev --ignore-not-found
```

**Delete the files on the node and the laptop**
```bash
rm -rf ~/rbac-users
rm ~/.kube/hans-adm.conf ~/.kube/hans-dev.conf
```

---