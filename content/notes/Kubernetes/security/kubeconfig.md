---
title: Kubeconfig
date: 2026-05-10
weight: 6
tags: [Security, KubeConfig]
---
# Kubeconfig

A kubeconfig file is a YAML-formatted configuration file that stores the authentication and connection details that `kubectl` needs to access Kubernetes clusters. It organizes cluster endpoints, user credentials, and context definitions in a single file, typically located at `~/.kube/config`.

---

## The Problem: Repetitive Authentication Flags

Without a `kubeconfig` file, every `kubectl` command requires explicit flags for the server address, client certificate, client key, and certificate authority:

```bash
kubectl get pods \
  --server https://my-kube-playground:6443 \
  --client-key admin.key \
  --client-certificate admin.crt \
  --certificate-authority ca.crt
```

Moving this information into a `kubeconfig` file eliminates the need to pass these flags on every command.

---

## Default Location

By default, `kubectl` looks for a file named `config` at:

```
~/.kube/config
```

If the file exists there, `kubectl` reads it automatically. A custom file can be specified using the `--kubeconfig` flag:

```bash
kubectl get pods --kubeconfig /path/to/my-config
```

To set a different file as the default permanently:

```bash
echo "export KUBECONFIG=~/my-kube-config" >> ~/.bashrc
source ~/.bashrc
```

---

## Structure of the KubeConfig File

The `kubeconfig` file has three top-level sections: `clusters`, `users`, and `contexts`.

```yaml
apiVersion: v1
kind: Config
current-context: my-kube-admin@my-kube-playground

clusters:
  - name: my-kube-playground
    cluster:
      server: https://my-kube-playground:6443
      certificate-authority: /etc/kubernetes/pki/ca.crt

users:
  - name: my-kube-admin
    user:
      client-certificate: /etc/kubernetes/pki/admin.crt
      client-key: /etc/kubernetes/pki/admin.key

contexts:
  - name: my-kube-admin@my-kube-playground
    context:
      cluster: my-kube-playground
      user: my-kube-admin
```

### Clusters

Defines the Kubernetes clusters to access. Each entry specifies the API server address and the certificate authority used to verify it. A single file can define multiple clusters across different environments, organizations, or cloud providers.

### Users

Defines the user accounts used to authenticate against those clusters. Different users may have different privileges on different clusters.

### Contexts

Contexts bind a user to a cluster. A context named `admin@production` tells `kubectl` to use the `admin` user credentials when connecting to the `production` cluster. Contexts do not create new users or configure authorization. They define which existing credentials are used to access which cluster.

### current-context

The `current-context` field sets the default context `kubectl` uses when no context is specified explicitly:

```yaml
current-context: dev-user@google
```

---

## Namespaces in Contexts

A context can be configured to operate in a specific namespace automatically by adding a `namespace` field:

```yaml
contexts:
  - name: dev-user@dev-cluster
    context:
      cluster: dev-cluster
      user: dev-user
      namespace: development
```

When this context is active, `kubectl` operates within the `development` namespace without requiring the `-n` flag.

**Switch from Namespace**
```bash
kubectl config set-context --current --namespace=prod
```

---

## Managing KubeConfig with kubectl config

`kubectl` provides subcommands to inspect and modify the `kubeconfig` file without editing YAML directly. All changes are written back to the file immediately.

**View the current configuration**
```bash
kubectl config view
```

**Switch the active context**
```bash
kubectl config use-context prod-user@production
```

**Verify the active context**
```bash
kubectl config get-contexts
```

**Add a new cluster entry**
```bash
kubectl config set-cluster production \
  --server=https://production-cluster:6443 \
  --certificate-authority=/etc/kubernetes/pki/prod-ca.crt
```

**Add a new user entry**
```bash
kubectl config set-credentials prod-admin \
  --client-certificate=/etc/kubernetes/pki/prod-admin.crt \
  --client-key=/etc/kubernetes/pki/prod-admin.key
```

**Create a context linking a cluster and a user**
```bash
kubectl config set-context prod-admin@production \
  --cluster=production \
  --user=prod-admin \
  --namespace=default
```

---

## Embedding Certificates

Certificate files can be referenced by absolute path or embedded directly as Base64-encoded content. Absolute paths are preferred for local setups. Embedded certificates are useful for self-contained config files that do not depend on files being present at a specific path.

**Reference by path**
```yaml
certificate-authority: /etc/kubernetes/pki/ca.crt
```

**Embed as Base64**
```yaml
certificate-authority-data: LS0tLS1CRUdJTi...
```

**Encode a certificate for embedding**
```bash
base64 -w 0 /etc/kubernetes/pki/ca.crt
```

**Decode an embedded certificate for inspection**
```bash
echo "LS0tLS1CRUdJTi..." | base64 --decode
```

The same pattern applies to `client-certificate-data` and `client-key-data` in the users section.

---

## Setting Up a KubeConfig File from Scratch

**Create the `.kube` directory**
```bash
mkdir -p ~/.kube
```

**Write the kubeconfig file**
```bash
cat <<EOF > ~/.kube/config
apiVersion: v1
kind: Config

clusters:
  - name: my-kube-playground
    cluster:
      server: https://my-kube-playground:6443
      certificate-authority: /etc/kubernetes/pki/ca.crt

users:
  - name: my-kube-admin
    user:
      client-certificate: /etc/kubernetes/pki/admin.crt
      client-key: /etc/kubernetes/pki/admin.key

contexts:
  - name: my-kube-admin@my-kube-playground
    context:
      cluster: my-kube-playground
      user: my-kube-admin

current-context: my-kube-admin@my-kube-playground
EOF
```

> Use absolute paths for all certificate files to avoid resolution issues when `kubectl` is run from different working directories.

**Verify the configuration**
```bash
kubectl config view
kubectl get nodes
```

---

## Key Facts

- The kubeconfig file does not create Kubernetes objects. It is a client-side configuration file only.
- Each section (`clusters`, `users`, `contexts`) is an array, so multiple entries of each type can coexist in a single file.
- Clusters, users, and contexts can be added imperatively with `kubectl config set-cluster`, `set-credentials`, and `set-context` without editing YAML directly.

---
