---
title: Image Registries
date: 2026-03-16
weight: 3
tags: [Images, Pods, Containers]
---
# Image Registries

A registry is a centralised storage server for container images. Understanding image naming conventions and how Kubernetes authenticates against private registries is essential for managing workloads securely.

---

## Image Naming Conventions

Every pod definition references at least one container image:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

The image name `nginx` is shorthand for a fully qualified reference:

```
docker.io/library/nginx
```

Three components:

- **Registry**: `docker.io` is Docker Hub, the default when none is specified.
- **Account**: `library` is the default account for Docker's official images.
- **Image name**: `nginx` is the actual image name.

When publishing under a custom account:

```
docker.io/my-company/my-app
```

---

## Registries

A registry is a centralised storage server for pushing and pulling container images. When Kubernetes deploys a pod, the container runtime on the worker node pulls the image from the registry specified in the pod definition.

Commonly used public registries:

- **Docker Hub** (`docker.io`): the default public registry.
- **Google Container Registry** (`gcr.io`): hosts many Kubernetes-related images.

---

## Private Registries

For in-house applications that should not be publicly accessible, a private registry is the appropriate solution. Major cloud providers offer managed private registries: AWS Elastic Container Registry (ECR), Azure Container Registry (ACR), and Google Artifact Registry. Self-hosted options include Harbor and Nexus.

### Creating an Image Pull Secret

Kubernetes passes registry credentials to the container runtime using a Secret of type `docker-registry`:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.example.com \
  --docker-username=my-user \
  --docker-password=my-password \
  --docker-email=my-user@example.com
```

**Verify the secret was created:**

```bash
kubectl get secret regcred
kubectl describe secret regcred
```

### Referencing the Secret in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
    - name: my-app
      image: private-registry.example.com/my-company/my-app:latest
  imagePullSecrets:
    - name: regcred
```

When Kubernetes schedules the pod, the kubelet reads the `regcred` Secret and uses its credentials to authenticate with the private registry before pulling the image.

### Attaching an Image Pull Secret to a Service Account

Rather than adding `imagePullSecrets` to every pod definition individually, the secret can be attached to a Service Account. All pods using that Service Account automatically inherit the pull credentials:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

**Verify the Service Account has the secret attached:**

```bash
kubectl describe serviceaccount default
```

### Using Multiple Registries

A pod can reference images from multiple private registries by listing multiple secrets under `imagePullSecrets`:

```yaml
imagePullSecrets:
  - name: regcred-aws
  - name: regcred-gcr
```

Each secret holds credentials for a different registry. The kubelet tries each in order when pulling images.

### Verifying Image Pull Credentials

If a pod fails to pull an image, the event log shows the reason:

```bash
kubectl describe pod my-app-pod
```

Look for events such as:

```
Failed to pull image: unauthorized: authentication required
```

or:

```
ErrImagePull / ImagePullBackOff
```

These indicate missing or incorrect credentials. Check that the secret exists in the same namespace as the pod and that the correct server address is specified.

---

## Reference

| Concept                        | Detail                                     |
|--------------------------------|--------------------------------------------|
| Default registry               | `docker.io` (Docker Hub)                   |
| Default account                | `library` (official images)                |
| Full image reference format    | `<registry>/<account>/<image>:<tag>`       |
| Private registry use case      | Internal or proprietary application images |
| Credential mechanism           | `kubernetes.io/dockerconfigjson` Secret    |
| Pod field for pull credentials | `imagePullSecrets`                         |
| Service Account attachment     | `kubectl patch serviceaccount`             |