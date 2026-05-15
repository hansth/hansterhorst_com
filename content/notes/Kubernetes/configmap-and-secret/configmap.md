---
title: Configmap
date: 2026-03-16
weight: 1
tags: [ConfigMap]
---
# Kubernetes ConfigMaps

A **ConfigMap** is a Kubernetes resource used to store **non-confidential** configuration data as key-value pairs. It separates configuration from container images, making applications portable across environments (dev, staging, production) without rebuilding images.

Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files mounted in a volume.

> **Note**: ConfigMaps are for non-sensitive data. For sensitive data like passwords, tokens, and keys, use **Secrets** instead.

---

## Environment Variables

### ENV in Docker

In Docker, environment variables are specified with the `-e` or `--env` flags:

```bash
docker run -e STAGE=dev ubuntu
docker run --env STAGE=dev ubuntu
```

### ENV in Kubernetes

In Kubernetes, environment variables are specified with the `--env` flag:

```bash
kubectl run env-pod --image=ubuntu --env=STAGE=dev --env=TEST=yes
```

In a Pod manifest, use the `env` field and define variables as a list:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
  labels:
    run: env-pod
spec:
  containers:
    - name: env-pod
      image: ubuntu
      env:
        - name: STAGE
          value: dev
        - name: TEST
          value: "yes"
```
- `env`: Field name.
- `name`: Name of the variable.
- `value`: Value of the variable.

---

## Creating ConfigMaps

When managing multiple environment variables, define them in a ConfigMap resource. There are two approaches.

### Imperative

These commands are fast and don't require YAML files, making them ideal for quick tasks. However, they're limited; complex setups, such as multiple containers or ENV variables, require long commands, and they don't leave a durable record, making it hard to reproduce.

**Create from literals**
```bash
kubectl create configmap imperative-input-config \
  --from-literal=STAGE=dev \
  --from-literal=TEST=yes
```
- `--from-literal`: Specify key-value pairs directly from the command line.

**Verify the ConfigMap**
```bash
kubectl get configmaps imperative-input-config -o yaml
```
```yaml
apiVersion: v1
data:
  STAGE: dev
  TEST: "yes"
kind: ConfigMap
metadata:
  name: imperative-input-config
  namespace: default
```

**Create from a file**
```bash
kubectl create configmap imperative-file-config \
  --from-file=app_config.properties
```
- `--from-file`: Read key-value pairs from a file.

**Verify the ConfigMap**
```bash
kubectl get configmaps imperative-file-config -o yaml
```
```yaml
apiVersion: v1
data:
  app_config.properties: |
    STAGE=prod
    TEST=no
kind: ConfigMap
metadata:
  name: imperative-file-config
  namespace: default
```

A practical shortcut for creating a properties file using a heredoc:

```bash
cat << EOF > config-testing.properties
APP_ENV=test
LOG_LEVEL=error
EOF
```

Create the ConfigMap from the file using `--from-env-file`:

```bash
kubectl create configmap app-config-testing \
  --from-env-file=config-testing.properties \
  --dry-run=client -o yaml > app-config-testing.yaml
```

**Apply the ConfigMap**
```bash
kubectl apply -f app-config-testing.yaml
```

### Declarative

With the declarative approach, you define the ConfigMap in a manifest file, manage it with version control, and apply changes, so Kubernetes reconciles the desired state with the current state in the cluster.

**ConfigMap manifest**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: declarative-config
data:
  STAGE: dev
  TEST: "yes"
```

**Generate a manifest using dry-run**
```bash
kubectl create configmap declarative-config \
  --from-literal=STAGE=dev \
  --from-literal=TEST=yes \
  --dry-run=client -o yaml > declarative-config.yaml
```

**Apply the manifest**
```bash
kubectl apply -f declarative-config.yaml
```

**Verify the ConfigMap**
```bash
kubectl get configmaps declarative-config
```
```
NAME                 DATA   AGE
declarative-config   2      13s
```

---

## Using ConfigMaps in Pods

There are three ways to inject ConfigMap data into a Pod.

### 1. All Keys with `envFrom`

Inject all ConfigMap key-value pairs as environment variables at once:

```yaml
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      envFrom:
        - configMapRef:
            name: declarative-config
```

> **Note**: `configMapRef.name` references the ConfigMap's `metadata.name`, not a filename.

Updating the ConfigMap and redeploying Pods automatically applies the new configuration without modifying the Pod manifest.

### 2. Single Key with `configMapKeyRef`

Inject a specific key from a ConfigMap, optionally renaming the environment variable:

```yaml
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      env:
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config-testing
              key: APP_ENV
```

This injects only `APP_ENV` from the ConfigMap, exposed as `ENVIRONMENT` inside the container.

### 3. Volume Mount

Mount a ConfigMap as files inside the container. Each key becomes a file whose value is its content. Useful for configuration files that applications read from disk:

```yaml
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config-testing
```

This creates `/etc/config/APP_ENV` (containing `test`) and `/etc/config/LOG_LEVEL` (containing `error`) inside the container.

To mount specific keys only, use the `items` field:

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config-testing
      items:
        - key: APP_ENV
          path: environment.conf
```

This mounts only `APP_ENV` as `/etc/config/environment.conf`.

---

## Immutable ConfigMaps

Since Kubernetes 1.21, ConfigMaps can be made immutable, preventing any changes to their data once created:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-testing
data:
  APP_ENV: testing
  LOG_LEVEL: error
immutable: true
```

Attempting to modify an immutable ConfigMap results in an error:

```
Error: the ConfigMap "app-config-testing" is invalid:
  data: Forbidden: field is immutable when `immutable` is set
```

The API server rejects any modification to `data` or `binaryData` regardless of the method used. Changes to `metadata`, such as labels and annotations, are still allowed. To change and use the data in a Pod, delete and recreate the ConfigMap:

```bash
kubectl delete configmap app-config-testing
kubectl apply -f app-config-testing.yaml
```

---

## Update Behavior

- **Volume-mounted ConfigMaps**: Updated automatically when the ConfigMap changes, with some delay via the `kubelet` sync period.
- **Environment variables**: Not updated automatically. The Pod must be restarted to pick up changes.
- **Immutable ConfigMaps**: Cannot be updated. Delete and recreate to change values.

---

## ConfigMap vs Secret

| Aspect                 | ConfigMap                   | Secret                       |
|------------------------|-----------------------------|------------------------------|
| **Purpose**            | Non-sensitive config        | Sensitive data               |
| **Data storage**       | Plain text                  | Base64-encoded               |
| **Encryption at rest** | No (by default)             | Can be configured            |
| **Use cases**          | App settings, feature flags | Passwords, tokens, TLS certs |
| **Size limit**         | 1 MiB                       | 1 MiB                        |

---

## Notes

- **Namespace-scoped**: ConfigMaps exist within a Namespace. A Pod can only reference ConfigMaps in its own Namespace.
- **Size limit**: ConfigMaps are limited to 1 MiB of data.
- **Optional references**: Mark a reference as `optional: true` so the Pod starts even if the ConfigMap does not exist.

---

## Common `kubectl` Commands

**List ConfigMaps**
```bash
kubectl get configmaps
```

**Inspect a ConfigMap**
```bash
kubectl describe configmap app-config
```

**View YAML output**
```bash
kubectl get configmap app-config -o yaml
```

**Edit a ConfigMap**
```bash
kubectl edit configmap app-config
```

**Delete a ConfigMap**
```bash
kubectl delete configmap app-config
```

---

## Conclusion

ConfigMaps are the standard mechanism for externalizing configuration from container images.

They support three consumption methods:

- Injecting all keys at once with `envFrom`.
- Selecting a single key with `configMapKeyRef`.
- Mounting keys as files via volumes.

Environment variables require a Pod restart to pick up changes, while volume-mounted ConfigMaps update automatically. Immutable ConfigMaps prevent accidental changes and are useful for stable, production configurations. ConfigMaps are namespace-scoped and limited to 1 MiB of data.

---
