---
title: Encryption at Rest
date: 2026-03-14
weight: 3
tags: [Encryption, etcd, Secret, Security]
---
# Secrets Encryption at Rest

In Kubernetes, Secrets are stored by default in `etcd` **without encryption**. Anyone with access to `etcd` can read Secret values in plain text.

> **Note**: See the official [Kubernetes documentation](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) for more information.

---

## How Secrets Are Stored

**Secrets** are Kubernetes resources designed to store **sensitive data**, such as passwords, tokens, SSH keys, TLS certificates, and other confidential information that requires careful handling.

**Create a Secret from literals**
```bash
kubectl create secret generic postgres-secret \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_USER=root \
  --from-literal=DB_PASS=super
```
- `generic`: Type of Secret for a local file, directory, or literal value.
- `--from-literal`: Literal input.

**Inspect the Secret**
```bash
kubectl get secret postgres-secret -o yaml
```
```yaml
apiVersion: v1
data:
  DB_HOST: cG9zdGdyZXM=
  DB_PASS: c3VwZXI=
  DB_USER: cm9vdA==
kind: Secret
metadata:
  creationTimestamp: "2026-03-14T08:52:06Z"
  name: postgres-secret
  namespace: dev
  resourceVersion: "793120"
  uid: 8eb24d76-f7c5-4d19-8533-6e94ea66ef14
type: Opaque
```

The three data keys are encoded in base64 format. Base64 encoding is **not encryption**. Anyone can decode these values:

```bash
echo "c3VwZXI=" | base64 --decode
# super
```

> **Note**: Never commit Secret manifest files to version control.

---

## Secret Data in `etcd`

Secrets are stored **unencrypted** in `etcd`. To inspect how a Secret is stored, install the `etcd-client` on the control plane node:

```bash
sudo apt update && sudo apt install etcd-client
```

**Read the Secret from `etcd`**
```bash
sudo ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/postgres-secret | hexdump -C
```
- `| hexdump -C`: Displays the content in hexadecimal format for readability.

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 70 6f 73 74 67 72  |s/default/postgr|
00000020  65 73 2d 73 65 63 72 65  74 0a 6b 38 73 00 0a 0c  |es-secret.k8s...|
00000030  0a 02 76 31 12 06 53 65  63 72 65 74 12 9b 02 0a  |..v1..Secret....|
00000040  d8 01 0a 0f 70 6f 73 74  67 72 65 73 2d 73 65 63  |....postgres-sec|
00000050  72 65 74 12 00 1a 07 64  65 66 61 75 6c 74 22 00  |ret....default".|
00000120  5f 48 4f 53 54 12 08 70  6f 73 74 67 72 65 73 12  |_HOST..postgres.|
00000130  10 0a 07 44 42 5f 50 41  53 53 12 05 73 75 70 65  |...DB_PASS..supe|
00000140  72 12 0f 0a 07 44 42 5f  55 53 45 52 12 04 72 6f  |r....DB_USER..ro|
00000150  6f 74 1a 06 4f 70 61 71  75 65 1a 00 22 00 0a     |ot..Opaque.."..|
```

The Secret values (`postgres`, `super`, `root`) are visible in plain text, confirming that the data is stored unencrypted.

---

## Setting Up Encryption at Rest

### Checking If Encryption Is Already Enabled

Check whether the `--encryption-provider-config` flag is set on the `kube-apiserver`:

```bash
ps aux | grep kube-apiserver | grep encryption-provider-config
```

In a `kubeadm` setup, inspect the static Pod manifest directly:

```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption-provider-config
```

If neither command returns a result, encryption at rest is not enabled.

### Creating the Encryption Manifest File

To enable **encryption at rest**, you have to create an encryption manifest file and reference it to the `kube-apiserver` by adding the `--encryption-provider-config` option.

#### Understanding the EncryptionConfiguration

The `EncryptionConfiguration` manifest tells the `kube-apiserver` how to encrypt and decrypt resources before storing them in `etcd`.

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - identity: {}       # plain text, no encryption
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
      - secretbox:
          keys:
            - name: key1
              secret: YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY=
```

Providers are listed in priority order. The first provider is used for encryption; any provider on the list can be used for decryption. If `identity` is listed first, no encryption occurs.

Available providers:

| Provider    | Description                                 |
|-------------|---------------------------------------------|
| `identity`  | No encryption. Resources are written as-is. |
| `aesgcm`    | AES in GCM mode.                            |
| `aescbc`    | AES in CBC mode.                            |
| `secretbox` | XSalsa20 and Poly1305.                      |

To encrypt data, place a real encryption provider first and `identity` last. The `identity` provider at the bottom ensures existing unencrypted Secrets can still be read.

### Generating a Key and Creating the Encryption Manifest

**Generate a 32-byte random base64-encoded key**
```bash
head -c 32 /dev/urandom | base64
# ZDnqW7QayE3fl86+CLlDOV1wbCWdcOGHpUyEGaKdlv0=
```

**Create the encryption manifest `etcd-encryption.yaml` on the control plane node**
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ZDnqW7QayE3fl86+CLlDOV1wbCWdcOGHpUyEGaKdlv0=
      - identity: {}
```

`aescbc` is the first provider, so it is used for encryption. The `identity` provider at the bottom allows existing unencrypted Secrets to still be read during the transition period.

---

## Configuring the `kube-apiserver`

### Place the Configuration File

Create a directory on the Control Plane Node to store the encryption manifest and add the manifest to it.

```bash
sudo mkdir -p /etc/kubernetes/encryptions && \
  sudo mv etcd-encryption.yaml /etc/kubernetes/encryptions/
```

### Back Up the `kube-apiserver` Manifest

First, create a **backup** directory for the `kube-apiserver` manifest file.

```bash
sudo mkdir -p /etc/kubernetes/backups

sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml \
  /etc/kubernetes/backups/kube-apiserver.yaml.$(date +%Y-%m-%d)-backup
```

> **Note**: Do not store backup files in `/etc/kubernetes/manifests/`. The `kubelet` monitors this directory and processes all files within it.

### Edit the `kube-apiserver` Manifest

Open `/etc/kubernetes/manifests/kube-apiserver.yaml` and make the following three changes:

**1. Add the encryption provider config flag** to the command arguments
```yaml
- --encryption-provider-config=/etc/kubernetes/encryptions/etcd-encryption.yaml
```

**2. Add a volume mount** to make the encryption file accessible inside the Pod
```yaml
volumeMounts:
  - name: encryption
    mountPath: /etc/kubernetes/encryptions
    readOnly: true
```

**3. Add the corresponding volume** definition
```yaml
volumes:
  - name: encryption
    hostPath:
      path: /etc/kubernetes/encryptions
      type: DirectoryOrCreate
```

This maps `/etc/kubernetes/encryptions` on the host into the `kube-apiserver` Pod, making the encryption manifest available at the path specified by the flag.

> **Note**: In a multi-control-plane cluster, these changes must be applied to all control plane nodes. If any `kube-apiserver` lacks encryption configuration, it will write Secrets to `etcd` unencrypted.

---

## Waiting for the `kube-apiserver` to Restart

After saving the manifest, the `kube-apiserver` Pod restarts automatically. Monitor the restart with `crictl`:

```bash
sudo crictl pods
```

```
POD ID         CREATED         STATE     NAME
72668d705a314  9 minutes ago   NotReady  kube-apiserver-k8s-master-1
d2336de354db1  25 seconds ago  Ready     kube-apiserver-k8s-master-1
```

Once the Pod is back in `Ready` state, verify the flag is active:

```bash
ps aux | grep kube-apiserver | grep encryption-provider-config
```

You can also inspect the running Pod manifest to confirm the volume and flag are present:

```bash
kubectl -n kube-system get pod kube-apiserver-k8s-master-1 -o yaml
```

This should return a result confirming the option is configured.

```yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - ...
    - --encryption-provider-config=/etc/kubernetes/encryptions/etcd-encryption.yaml
    image: registry.k8s.io/kube-apiserver:v1.31.14
    ...
    volumeMounts:
    - ...
    - name: encryption
      mountPath: /etc/kubernetes/encryptions
      readOnly: true
  hostNetwork: true
  ...
  volumes:
  - ...
  - name: encryption
    hostPath:
      path: /etc/kubernetes/encryptions
      type: DirectoryOrCreate
```

---

## Verifying Encryption Is Working

**Create a new Secret after encryption has been enabled**
```bash
kubectl create secret generic test-encrypted-secret \
  --from-literal=test-key=topsecretvalue
```

**Read the new Secret from `etcd`**
```bash
sudo ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-encrypted-secret | hexdump -C | tail
```
```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 74 65 73 74 2d 65  |s/default/test-e|
00000020  6e 63 72 79 70 74 65 64  2d 73 65 63 72 65 74 0a  |ncrypted-secret.|
00000030  6b 38 73 3a 65 6e 63 3a  61 65 73 63 62 63 3a 76  |k8s:enc:aescbc:v|
00000040  31 3a 6b 65 79 31 3a 1a  03 13 43 39 71 46 21 cd  |1:key1:...C9qF!.|
```

The Secret value is no longer visible in plain text. The prefix `k8s:enc:aescbc:v1:key1:` confirms that encryption is active using the provider specified in the manifest.

**Check the original Secret created before encryption was enabled**
```bash
sudo ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/postgres-secret | hexdump -C | tail
```
```
00000120  5f 48 4f 53 54 12 08 70  6f 73 74 67 72 65 73 12  |_HOST..postgres.|
00000130  10 0a 07 44 42 5f 50 41  53 53 12 05 73 75 70 65  |...DB_PASS..supe|
00000140  72 12 0f 0a 07 44 42 5f  55 53 45 52 12 04 72 6f  |r....DB_USER..ro|
00000150  6f 74 1a 06 4f 70 61 71  75 65 1a 00 22 00 0a     |ot..Opaque.."..|
```

The old values (`postgres`, `super`, `root`) are still visible. Encryption only applies to newly created or updated objects. Existing Secrets are not automatically re-encrypted.

---

## Encrypting All Existing Secrets

To ensure that all existing Secrets stored in `etcd` are encrypted, you must read each Secret and then replace it. This read-and-replace operation triggers re-encryption:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

This retrieves all Secrets and replaces them with the same data, which causes the API server to write them back to `etcd` using the active encryption provider.

> **_Note_**: Once all Secrets are confirmed to be re-encrypted, `identity` can be removed from the provider's list in the encryption manifest. After restarting the `kube-apiserver`, any Secret still stored in `etcd` in an unencrypted format will become unreadable. Remove `identity` only when you are confident all Secrets have been re-encrypted.

---

