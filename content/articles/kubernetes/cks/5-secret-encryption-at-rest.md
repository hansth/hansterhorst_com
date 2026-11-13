---
title: Secret Encryption at Rest
date: 2026-11-05
weight: 5
draft: false
reviewed: true
tags:
  - CKS
  - Security
  - Kubernetes
  - Secrets
  - etcd
  - Tutorial
---
# Project Secret Encryption at Rest

In this project, I enabled encryption at rest to secure Secrets stored in etcd on my homelab cluster. Before the change, etcd stored every Secret in plain text. Anyone with access to etcd, its data directory, or an etcd backup could read Secrets in the cluster. After the change, the API server encrypts each Secret before writing it to etcd.

In Kubernetes, a Secret resource is used to store sensitive data such as passwords, tokens, and keys. The Secret is stored by default in `etcd` **without encryption**, meaning anyone with access to `etcd` can read the Secret in plain text.

> **_Note:_** See the official [Kubernetes documentation](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) for more information.

---

## The Plan

The cluster has three control plane nodes. Each node runs its own `kube-apiserver` and its own etcd instance. Every API server needs the same encryption configuration with the same key. Without it, an API server writes Secrets in plain text and can't read Secrets encrypted by the others.

I first prove that etcd stores Secrets in plain text. Then I create one configuration file, copy it to the three control plane nodes, and change the API servers one at a time. HAProxy sends the API requests to the other two nodes while one API server restarts.

The project uses these names and values:

| Item               | Value                                                                          |
| ------------------ | ------------------------------------------------------------------------------ |
| Provider           | `aescbc`, with `identity` as the fallback to read old data                     |
| Key name           | `key1`                                                                         |
| Configuration file | `/etc/kubernetes/encryptions/etcd-encryption.yaml` on every control plane node |
| Manifest backup    | `/etc/kubernetes/backups/` on every control plane node                         |
| Test Secrets       | `postgres-secret` before the change, `test-encrypted-secret` after it          |

> **_Note:_** The Kubernetes docs advise against `aescbc` for new clusters and recommend a `kms` v2 provider. I use `aescbc` because the CKS exam tasks use it, and a KMS needs an external key service.

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
  creationTimestamp: "2026-11-12T08:52:06Z"
  name: postgres-secret
  namespace: dev
  resourceVersion: "793120"
  uid: 8eb24d76-f7c5-4d19-8533-6e94ea66ef14
type: Opaque
```

The three data keys are encoded in base64 format. Base64 encoding is **not encryption**. Anyone can decode these values:

```bash
echo "c3VwZXI=" | base64 --decode
```
```
super
```

> **_Note:_** Never commit Secret manifest files to version control such as Git.

---

## Step 1: Check That Encryption Is Off

The `--encryption-provider-config` flag of the `kube-apiserver` turns on encryption at rest. Check if the flag is already set.

```bash
ps -ef | grep kube-apiserver | grep encryption-provider-config
```
- `-ef`: Get every running process (`e`) with extra information (`f`).

In a `kubeadm` setup, inspect the static Pod manifest directly:

```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption-provider-config
```

If both commands don't return a result, encryption at rest is not enabled.

---

## Step 2: Read a Secret From etcd

By default, etcd stores Secrets **without encryption**. Anyone with access to etcd can read these values.

To see how etcd stores a Secret, `exec` into the etcd Pod and run `etcdctl` inside the container.

**List the etcd Pods**
```bash
kubectl -n kube-system get pods -l component=etcd
```
```
NAME                READY   STATUS    RESTARTS      AGE
etcd-k8s-master-1   1/1     Running   6 (10d ago)   155d
etcd-k8s-master-2   1/1     Running   5 (10d ago)   155d
etcd-k8s-master-3   1/1     Running   6 (10d ago)   155d
```

To read the Secret from etcd, run the following command to connect with the certificates located in the `/etc/kubernetes/pki/etcd/` directory.

**Read the Secret from `etcd`**
```bash
kubectl -n kube-system exec etcd-k8s-master-1 -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/postgres-secret | hexdump -C
```
- `hexdump -C`: Displays the content in hexadecimal format for readability.

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 70 6f 73 74 67 72  |s/default/postgr|
00000020  65 73 2d 73 65 63 72 65  74 0a 6b 38 73 00 0a 0c  |es-secret.k8s...|
00000030  0a 02 76 31 12 06 53 65  63 72 65 74 12 9b 02 0a  |..v1..Secret....|
00000040  d8 01 0a 0f 70 6f 73 74  67 72 65 73 2d 73 65 63  |....postgres-sec|
00000050  72 65 74 12 00 1a 07 64  65 66 61 75 6c 74 22 00  |ret....default".|
...
00000120  5f 48 4f 53 54 12 08 70  6f 73 74 67 72 65 73 12  |_HOST..postgres.|
00000130  10 0a 07 44 42 5f 50 41  53 53 12 05 73 75 70 65  |...DB_PASS..supe|
00000140  72 12 0f 0a 07 44 42 5f  55 53 45 52 12 04 72 6f  |r....DB_USER..ro|
00000150  6f 74 1a 06 4f 70 61 71  75 65 1a 00 22 00 0a     |ot..Opaque.."..|
```

The Secret values (`postgres`, `super`, `root`) are visible in plain text, confirming that the data is stored unencrypted.

---

## Step 3: Create the Encryption Configuration

To enable **encryption at rest**, you have to create an encryption manifest file and reference it to the `kube-apiserver` by adding the `--encryption-provider-config` option.

### Understanding the EncryptionConfiguration

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

Each provider (except `identity`) requires a secret key for the encryption algorithm.

### Creating the Encryption Manifest File

The `EncryptionConfiguration` tells the API server which resources to encrypt and with which provider. The first provider in the list encrypts new data. The API server tries the providers in order when it reads data.

**Generate a 32-byte key on the laptop**
```bash
head -c 32 /dev/urandom | base64
```
```
KsqN7S+Gsuye4TiSdTWbHdEz0mTgmW39iesf65biUIM=
```
The output is the key. `aescbc` accepts a key of 16, 24, or 32 bytes. A 32-byte key is the strongest. Keep the key out of notes, screenshots, and Git.

The configuration file encrypts only Secrets, with `aescbc` first:
```bash
cat <<'EOF' > /tmp/etcd-encryption.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <ENCRYPTION_KEY>
      - identity: {}
EOF
```
- `resources.resources`: The resources to encrypt. Only Secrets hold sensitive data in this cluster.
- `aescbc`: The first provider. It encrypts every new Secret.
- `keys.name`: The name of the key. The API server writes it in the prefix of the data, `k8s:enc:aescbc:v1:key1:`.
- `keys.secret`: The base64-encoded key from the previous command.
- `identity: {}`: The fallback. It reads the Secrets that etcd still stores in plain text.

> **_Note:_** The configuration file is stored in the `/tmp` directory. The system removes it at a reboot, or after a few days on macOS.

---

## Step 4: Copy the Configuration to the Control Plane Nodes

All three API servers need the same file with the same key. A static Pod can only read files from the node, and the API server reads this file at startup.

**Copy the file to every control plane node**
```bash
for node in master-1 master-2 master-3; do
  scp /tmp/etcd-encryption.yaml $node@k8s-$node:/tmp/
  ssh -t $node@k8s-$node "sudo mkdir -p /etc/kubernetes/encryptions && \
    sudo mv /tmp/etcd-encryption.yaml /etc/kubernetes/encryptions/ && \
    sudo chown root:root /etc/kubernetes/encryptions/etcd-encryption.yaml && \
    sudo chmod 600 /etc/kubernetes/encryptions/etcd-encryption.yaml"
done
```
- `chmod 600`: Only root can read the key. The `kube-apiserver` runs as root in a kubeadm cluster.

**Delete the file on the laptop**
```bash
rm /tmp/etcd-encryption.yaml
```

---

## Step 5: Configure the API Servers

The kubelet restarts the `kube-apiserver` Pod when its manifest changes. I change one node at a time and wait until its API server is back. The other two API servers keep the cluster available behind HAProxy.

> **_Note:_** Between the first and the last node, the API servers have different configurations. An API server without the configuration can't read a Secret that another API server encrypted.

### Back Up the Manifest

A mistake in the manifest stops the API server on that node. A backup lets me roll back fast.

**Create the backup directory on the `k8s-master-1` node**
```bash
sudo mkdir -p /etc/kubernetes/backups
```

**Copy the kube-apiserver manifest**
```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml \
  /etc/kubernetes/backups/kube-apiserver.yaml.$(date +%Y-%m-%d)-backup
```

> **_Warning:_** Don't store the backup in `/etc/kubernetes/manifests/`. The kubelet runs every manifest in that directory as a static Pod.

### Edit the Manifest

The API server needs a flag and access to the configuration file. Open `/etc/kubernetes/manifests/kube-apiserver.yaml` and make three changes.

Add the flag to the `command` of the container:
```yaml
- --encryption-provider-config=/etc/kubernetes/encryptions/etcd-encryption.yaml
```

Add a `volumeMount` to the container:
```yaml
volumeMounts:
  - name: encryption
    mountPath: /etc/kubernetes/encryptions
    readOnly: true
```

Add the `volume` to the Pod spec:
```yaml
volumes:
  - name: encryption
    hostPath:
      path: /etc/kubernetes/encryptions
      type: DirectoryOrCreate
```
- `hostPath.path`: The directory on the node.
- `mountPath`: The same path inside the container. The flag points to the file at this path.

### Wait for the Restart

`kubectl` keeps working through HAProxy while the API server on this node restarts. `crictl` shows the Pod on the node itself.

**List the kube-apiserver Pods on the node**
```bash
sudo crictl pods --name kube-apiserver
```
```
POD ID         CREATED         STATE     NAME
72668d705a314  9 minutes ago   NotReady  kube-apiserver-k8s-master-1
d2336de354db1  25 seconds ago  Ready     kube-apiserver-k8s-master-1
```

The old Pod is `NotReady`, and the new Pod is `Ready`.

**Check the flag in the running Pod**
```bash
kubectl -n kube-system get pod kube-apiserver-k8s-master-1 -o yaml | grep encryption
```
```
- --encryption-provider-config=/etc/kubernetes/encryptions/etcd-encryption.yaml
    - mountPath: /etc/kubernetes/encryptions
      name: encryption
      path: /etc/kubernetes/encryptions
    name: encryption
```

Repeat the backup, the edit, and the checks on `k8s-master-2` and `k8s-master-3`. Change the Pod name in the last command to `kube-apiserver-k8s-master-2` and `kube-apiserver-k8s-master-3`.

---

## Step 6: Verify the Encryption

The API server only encrypts data that it writes after the restart. A new Secret shows if the encryption works.

**Create a new Secret**
```bash
kubectl create secret generic test-encrypted-secret \
  --from-literal=test-key=topsecretvalue
```

**Read the new Secret from etcd**
```bash
kubectl -n kube-system exec etcd-k8s-master-1 -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-encrypted-secret | hexdump -C
```
```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 74 65 73 74 2d 65  |s/default/test-e|
00000020  6e 63 72 79 70 74 65 64  2d 73 65 63 72 65 74 0a  |ncrypted-secret.|
00000030  6b 38 73 3a 65 6e 63 3a  61 65 73 63 62 63 3a 76  |k8s:enc:aescbc:v|
00000040  31 3a 6b 65 79 31 3a 1a  03 13 43 39 71 46 21 cd  |1:key1:...KsqN!.|
```

The name of the Secret is still visible because it's part of the etcd key. The prefix `k8s:enc:aescbc:v1:key1:` shows the provider and the key.

**Read the old Secret from etcd**
```bash
kubectl -n kube-system exec etcd-k8s-master-1 -- etcdctl \
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

The old Secret is still in plain text. The API server doesn't encrypt it until it writes the Secret again.

---

## Step 7: Encrypt the Existing Secrets

Every Secret from before the change is still in plain text, including the Secrets of Helm releases and the kubeadm bootstrap tokens. Rewriting a Secret makes the API server store it with the first provider.

**Rewrite all Secrets in all Namespaces**
```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

**Read the old Secret from etcd again**
```bash
kubectl -n kube-system exec etcd-k8s-master-1 -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/postgres-secret | hexdump -C
```
```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 70 6f 73 74 67 72  |s/default/postgr|
00000020  65 73 2d 73 65 63 72 65  74 0a 6b 38 73 3a 65 6e  |es-secret.k8s:en|
00000030  63 3a 61 65 73 63 62 63  3a 76 31 3a 6b 65 79 31  |c:aescbc:v1:key1|
00000040  3a be d3 ab d2 dd 5b cd  a4 78 cc e2 9a b2 1a f9  |:.....[..x......|
00000050  8a 6e 4c e6 a7 44 0a 7a  46 02 ac 3e b4 4d ce 6f  |.nL..D.zF..>.M.o|
...
00000170  f9 89 6e ed 34 31 6a af  4f 1f be a4 61 07 37 eb  |..n.41j.O...a.7.|
00000180  ef 84 67 a2 d5 d6 bb 48  ea 2d e2 a9 f7 2c ee 07  |..g....H.-...,..|
00000190  ce 0a                                             |..|
00000192
```

The output shows the prefix `k8s:enc:aescbc:v1:key1:`, and the values `postgres`, `super`, and `root` are gone.

> **_Warning:_** Keep `identity` in the list until every Secret is encrypted. Without it, the API server can't read a Secret that is still in plain text.

---

## Persist During Cluster Upgrades

The change in this project lives in the `kube-apiserver` manifest in `/etc/kubernetes/manifests/`. That file is **not** the source of truth on a kubeadm cluster. When kubeadm upgrades the nodes, it regenerates the manifest from the `ClusterConfiguration` in the `kubeadm-config` ConfigMap, and it removes the flag and the volume you added by hand.

To keep the change, add it to the `kubeadm-config` ConfigMap as well. kubeadm then writes it into the manifest on every cluster upgrade, on every control plane node.

**1. Back up the ConfigMap**
```bash
kubectl -n kube-system get cm kubeadm-config -o yaml | \
  sudo tee /etc/kubernetes/backups/kubeadm-config.yaml.$(date +%Y-%m-%d)-backup
```

**2. View the current `ClusterConfiguration`**
```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'
```

**3. Add the fixes**
```bash
kubectl -n kube-system edit cm kubeadm-config
```

Copy/paste the changes from the manifest file and add them to the `extraArgs` and `extraVolumes` under the `apiServer` section. Keep the fields that are already there.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
apiServer:
  extraArgs:
  #...
  # ETCD encryption at rest config
  - name: encryption-provider-config
    value: /etc/kubernetes/encryptions/etcd-encryption.yaml
  extraVolumes:
  #...
  # ETCD encryption at rest config
  - name: encryption
    hostPath: /etc/kubernetes/encryptions
    mountPath: /etc/kubernetes/encryptions
    readOnly: true
    pathType: Directory
#...
```
- `extraArgs`: Add the flag without the leading `--`.
- `value`: Always a string. Put numbers and booleans in double quotes.
- `extraVolumes`: Becomes both the `volumes` and `volumeMounts` entries in the manifest.

**4. Check what kubeadm would generate**
```bash
sudo kubeadm upgrade diff v1.35.5
```

This compares the current manifests with what kubeadm would generate for the next version update, without making any changes. If the output shows no lost flags or volumes, the ConfigMap matches the manifests.

> **_Note:_** kubeadm doesn't keep the comments in `kubeadm-config`. After an upgrade, the `# ETCD ...` comments are gone.

---

## Cleanup

In my homelab, I keep encryption at rest on. I only delete the test Secrets.

**Delete the test Secrets**
```bash
kubectl delete secret postgres-secret test-encrypted-secret
```

### Turn Off Encryption

To turn encryption off, first decrypt all data. Removing the flag first makes every encrypted Secret unreadable.

1. Move `identity: {}` to the top of the providers in the file on all three nodes. Keep `aescbc` below it to read the encrypted data.
2. Restart the API servers one at a time, as in step 5. The kubelet doesn't restart a static Pod when only the configuration file changes. Move the manifest out of `/etc/kubernetes/manifests/` and back to restart it.
3. Rewrite all Secrets, as in step 7. The API server now stores them in plain text.
4. Remove the flag, the volume, and the volume mount from the manifests, one node at a time.

---

## Common Errors

| Error                                                       | Cause                                                          | Fix                                                                                   |
| ----------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| The `kube-apiserver` Pod doesn't come back after the edit   | A wrong path, a missing volume, or a YAML error                | Read the logs in `/var/log/pods/kube-system_kube-apiserver-*/`, or restore the backup |
| The API server stops with an error about the key length     | The key isn't 16, 24, or 32 bytes                              | Generate the key again with `head -c 32 /dev/urandom \| base64`                       |
| A Secret can't be read on one API server                    | That API server has no configuration, or a different key       | Copy the same file to every control plane node                                        |
| Old Secrets are still in plain text in etcd                 | They weren't written after the change                          | Rewrite all Secrets, as in step 7                                                     |
| Secrets can't be read after you remove `identity`           | Some Secrets were still in plain text                          | Add `identity` back at the end of the list, and rewrite all Secrets                   |
---

## Conclusion

Encryption at rest protects the Secrets in etcd, in etcd backups, and on the disks of the control plane nodes. It doesn't change what a user with `get` rights on Secrets sees: `kubectl` still shows the decoded data. RBAC still decides who can read a Secret through the API.

The weak point of `aescbc` is the key. It's in plain text on every control plane node. An attacker with root on one control plane node can read the key and decrypt etcd. A `kms` v2 provider moves the key to an external key service. For my homelab, `aescbc` is enough. For production, I'd use KMS.

---

## Reference

| Concept                          | What it means                                                                |
| -------------------------------- | ---------------------------------------------------------------------------- |
| `--encryption-provider-config`   | The `kube-apiserver` flag that points to the `EncryptionConfiguration` file. |
| `EncryptionConfiguration`        | Lists the resources to encrypt and the providers in order.                   |
| First provider                   | Encrypts all new writes.                                                     |
| `identity`                       | No encryption. As the last provider, it reads old plain text data.           |
| `aescbc`                         | AES-CBC with a 32-byte key in the configuration file.                        |
| `kms` v2                         | Envelope encryption with a key in an external key service.                   |
| `k8s:enc:aescbc:v1:key1:`        | The prefix of an encrypted value in etcd: provider and key name.             |
| `kubectl replace` on all Secrets | Rewrites the Secrets, and the API server encrypts them.                      |

- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Using a KMS provider for data encryption](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/)
- [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- My notes: [Kubernetes Secrets](/notes/kubernetes/cks/3-minimize-microservice-vulnerabilities/5-1-kubernetes-secrets/) and [Encrypting Secret Data at Rest](/notes/kubernetes/cks/3-minimize-microservice-vulnerabilities/5-3-encrypting-secret-data-at-rest/).

---

## Best Practices

- Use the same configuration file and key on every control plane node.
- Change the API servers one at a time, and wait until each one is back.
- Back up the `kube-apiserver` manifest outside `/etc/kubernetes/manifests/` before you edit it.
- Restrict the configuration file to root with `chmod 600`.
- Keep the key out of Git, notes, and screenshots.
- Rewrite all Secrets after the change, and check one in etcd.
- Keep `identity` as the last provider until every Secret is encrypted.
- Use a `kms` v2 provider in production.

---

## CKS Exam Takeaways

- Check for encryption with `grep encryption-provider-config /etc/kubernetes/manifests/kube-apiserver.yaml`.
- The first provider encrypts. When `identity` is first, nothing is encrypted.
- `aescbc` needs a 32-byte key: `head -c 32 /dev/urandom | base64`.
- Add the flag, a `volumeMount`, and a `hostPath` volume to the `kube-apiserver` manifest.
- Verify with `etcdctl get /registry/secrets/<NAMESPACE>/<SECRET_NAME> | hexdump -C`, and look for `k8s:enc:aescbc:v1:`.
- Rewrite existing Secrets with `kubectl get secrets -A -o json | kubectl replace -f -`.

---
