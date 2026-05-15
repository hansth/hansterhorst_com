---
title: Secret
date: 2026-03-14 
weight: 2
tags: [Secrets, Security]
---
# Kubernetes Secrets

**Secrets** are Kubernetes resources designed to store **sensitive data**, such as passwords, tokens, SSH keys, TLS certificates, and other confidential information that requires careful handling.

With Secrets, you do not need to include sensitive data directly in application code, which improves security and maintainability. Secrets are **namespace-scoped**, meaning a Pod can only reference Secrets within the same Namespace.

---

## Critical Security Consideration

The most important thing to understand is that Kubernetes Secrets are **not encrypted**. They are simply **base64-encoded** and stored in the Kubernetes `etcd` database in an unencrypted form. Restricting access to `etcd` is therefore a fundamental security requirement, not an optional precaution.

### Understanding Base64 Encoding

With encryption, you need a decryption key to access the data. Encoding is used to preserve data usability, and you can decode the data by reversing the encoding algorithm.

**Encode a value**
```bash
echo -n "secret" | base64
# c2VjcmV0
```
- `echo -n`: Do not print the trailing newline character to avoid unexpected characters in the encoded value.

**Decode a value**
```bash
echo "c2VjcmV0" | base64 -d
# secret
```
- `-d`: Decode the value.

---

## Secrets vs ConfigMaps

Secrets are conceptually similar to ConfigMaps but specifically used for sensitive information.

| Aspect                | ConfigMap            | Secret              |
|-----------------------|----------------------|---------------------|
| Purpose               | Non-sensitive config | Sensitive data      |
| Encoding              | Plain text           | Base64 encoded      |
| Size limit            | 1 MiB                | 1 MiB               |
| tmpfs storage on node | No                   | Yes                 |
| Immutable option      | Yes                  | Yes                 |
| Volume mount updates  | Yes                  | Yes                 |
| Env var updates       | No (restart needed)  | No (restart needed) |

---

## Why Secrets Instead of ConfigMaps?

Sensitive data can be stored in ConfigMaps, but Secrets provide additional safeguards:

- **Intent signaling**: clearly marks data as sensitive.
- **Base64 encoding**: data is encoded at rest by default.
- **Access control**: RBAC can restrict Secret access more tightly than ConfigMaps.
- **Optional encryption at rest**: can be configured with `EncryptionConfiguration` on the API server.
- **Reduced exposure**: Secrets are only sent to Nodes that have Pods requiring them, and stored in `tmpfs` rather than written to disk on the Node.

---

## Secret Types

When creating a Secret imperatively, one of three types must be specified:

| Type            | `type` Field                          | Use Case                          |
|-----------------|---------------------------------------|-----------------------------------|
| Opaque          | `Opaque`                              | Default; arbitrary key-value data |
| Docker Registry | `kubernetes.io/dockerconfigjson`      | Image pull credentials            |
| TLS             | `kubernetes.io/tls`                   | TLS certificate and private key   |
| Basic Auth      | `kubernetes.io/basic-auth`            | Username and password             |
| SSH Auth        | `kubernetes.io/ssh-auth`              | SSH private key                   |
| Token           | `kubernetes.io/service-account-token` | Service account tokens (legacy)   |

The `kubectl` CLI types map to the manifest types as follows:

- `generic` → `Opaque`
- `docker-registry` → `kubernetes.io/dockerconfigjson`
- `tls` → `kubernetes.io/tls`

---

## Creating Secrets

### Imperative

With the imperative approach, values will be automatically base64-encoded before being stored in the `etcd` database.

**Create from literals**
```bash
kubectl create secret generic db-mysql-secret \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_USER=root \
  --from-literal=DB_PASS=password
```
- `generic`: Creates a Secret from a local file, directory, or literal value.

**Inspect the Secret**
```bash
kubectl describe secrets db-mysql-secret
```
```
Name:         db-mysql-secret
Namespace:    labs
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
DB_HOST:  5 bytes
DB_PASS:  8 bytes
DB_USER:  4 bytes
```

#### From a File

Create a Secret from a file by using the `--from-file` option.

`db_mysql_creds.yaml`
```yaml
DB_HOST: mysql
DB_USER: root
DB_PASS: password
```

**Create a Secret from a file**
```bash
kubectl create secret generic db-mysql-creds \
  --from-file=db_mysql_creds.yaml
```

Kubernetes encodes the file's contents in base64 and uses the filename as the key.

**Verify the Secret**
```bash
kubectl get secret db-mysql-creds -o yaml
```
```yaml
apiVersion: v1
data:
  db_mysql_creds.yaml: REJfSE9TVDogbXlzcWwKREJfVVNFUjogcm9vdApEQl9QQVNTOiBwYXNzd29yZAo=
kind: Secret
metadata:
  name: db-mysql-creds
  namespace: labs
type: Opaque
```

**Generate a Secret manifest using dry-run**
```bash
kubectl create secret generic credentials-secret \
  --from-literal=email=example@mail.com \
  --from-literal=password=secret \
  --dry-run=client -o yaml > credentials-secret.yaml
```

The generated manifest shows the values base64-encoded:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: credentials-secret
data:
  email: ZXhhbXBsZUBtYWlsLmNvbQ==
  password: c2VjcmV0
```

### Declarative

With the declarative approach, values in the `data` field must **first** be manually base64-encoded before being added to the manifest.

**Encode values with base-64**
```bash
echo -n "mysql" | base64
echo -n "example@mail.com" | base64
echo -n "secret" | base64
```

**Secret manifest**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-postgres-secret
data:
  DB_HOST: cG9zdGdyZXM=
  DB_PASS: cGFzc3dvcmQ=
  DB_USER: cm9vdA==
```

**Apply the manifest**
```bash
kubectl apply -f db-postgres-secret.yaml
```

#### From File

You can also create a Secret manifest by manually base64-encoding a file's contents and using the filename as the key.

`db_postgres_creds.yaml`
```yaml
DB_HOST: postgres
DB_USER: root
DB_PASS: password
```

**Encode the file content**
```bash
cat db_postgres_creds.yaml | base64
# REJfSE9TVDogcG9zdGdyZXMKREJfVVNFUjogcm9vdApEQl9QQVNTOiBwYXNzd29yZAo=
```

**Create a Secret manifest from a file**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-postgres-creds-secret
data:
  db_postgres_creds.yaml: REJfSE9TVDogcG9zdGdyZXMKREJfVVNFUjogcm9vdApEQl9QQVNTOiBwYXNzd29yZAo=
```

The key is the filename, and the encoded file contents are the value.

**Apply the manifest**

```bash
kubectl apply -f db-postgres-creds-secret.yaml
```

#### Using `stringData`

Instead of manually base64-encoding values, use `stringData` to provide plain text. Kubernetes encodes them automatically when the Secret is created:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: credentials-secret
type: Opaque
stringData:
  host: mysql
  user: root
  password: secret
```

The `stringData` field is write-only. When you retrieve a Secret, all values are shown under `data` in base64-encoded form. If a key is present in both `data` and `stringData`, the `stringData` value takes precedence.

---

## Immutable Secrets

Setting `immutable: true` prevents any modifications after creation. This protects against accidental updates and improves cluster performance by reducing API server load:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
immutable: true
data:
  password: c2VjcmV0
```

To update an immutable Secret, you have to delete and recreate it.

---

## Using Secrets in Pods

Once the Secrets are applied to the cluster, you can reference them in a resource, such as a Pod. There are different ways to do this.

### Environment Variables

**Inject all keys with `envFrom`**
```yaml
spec:
  containers:
    - name: postgres-pod
      image: nginx
      envFrom:
        - secretRef:
            name: db-postgres-secret
```

**Verify the environment variables**
```bash
kubectl exec postgres-pod1 -- env
```
```
DB_HOST=postgres
DB_PASS=password
DB_USER=root
```

**Inject a single key with `secretKeyRef`**
```yaml
spec:
  containers:
    - name: postgres-pod
      image: nginx
      env:
        - name: DB_EMAIL
          valueFrom:
            secretKeyRef:
              name: credentials-secret
              key: email
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: credentials-secret
              key: password
```

This maps Secret keys to custom environment variable names, useful when the application expects specific variable names that differ from the Secret keys.

### Volume Mount

Each key in the Secret becomes a separate file in the mount path. The file content is the decoded value:

```yaml
spec:
  containers:
    - name: postgres-pod
      image: nginx
      volumeMounts:
        - name: db-postgres-creds-volume
          mountPath: /var/secrets
          readOnly: true
  volumes:
    - name: db-postgres-creds-volume
      secret:
        secretName: db-postgres-creds-secret
```

**Inspect the mounted Secret file**
```bash
kubectl exec -it postgres-pod3 -- cat /var/secrets/db_postgres_creds.yaml
```
```
DB_HOST: postgres
DB_USER: root
DB_PASS: password
```

### Update Behavior

- **Volume-mounted Secrets**: updated automatically when the Secret changes, with a delay due to the `kubelet` sync period.
- **Environment variable Secrets**: Don't update automatically. The Pod must be recreated to pick up changes.

---

## Important Considerations

- Secrets are base64-encoded, not encrypted. Anyone with access to the Secret object can decode the data.
- Avoid committing Secret manifest files to version control.
- By default, Secrets stored in `etcd` are not encrypted. Consider enabling encryption at rest using `EncryptionConfiguration`.
- Restrict access to `etcd` and use RBAC to limit who can read Secrets in the cluster.

---

## Common `kubectl` Commands

**List Secrets**
```bash
kubectl get secrets
```

**Inspect a Secret**
```bash
kubectl describe secret credentials-secret
```

**View YAML output**
```bash
kubectl get secret credentials-secret -o yaml
```

**Decode a specific value**
```bash
kubectl get secret credentials-secret -o jsonpath='{.data.password}' | base64 -d
```

**Delete a Secret**
```bash
kubectl delete secret credentials-secret
```

---
## Conclusion

Kubernetes Secrets provide an effective way to store sensitive information securely within a cluster. While they offer a convenient way for managing sensitive data separately from application code, it’s important to remember a critical distinction: **Secrets are encoded, not encrypted**. This makes proper access control to the cluster’s `etcd` database essential.

### Critical Knowledge

Several key points about Secrets are essential:

**Encryption vs Encoding**: Secrets are not encrypted; they are encoded with base64. This is a crucial distinction that many practitioners overlook.

**Secret Types**: Pay attention to the type of Secret being used. For example, `Opaque` is the default type used when a type is not explicitly specified.

**Use Cases**: Understand the differences and appropriate use cases between Secrets, ConfigMaps, and Labels to choose the right configuration method for each scenario.

**TLS Secrets**: Be aware that there is a specific Secret type for TLS certificates, which will be covered in further study materials.

---
