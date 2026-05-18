---
title: Commands and Arguments
date: 2026-03-16
weight: 4
tags: [Arguments, Commands, Pods]
---
# Commands and Arguments

In container environments, **commands** define the executable binary that runs when a container starts, while **arguments** provide additional data, parameters, or flags to that executable.

## Docker vs. Kubernetes

The naming conventions differ between a Dockerfile and a Kubernetes Pod manifest, but they serve the same function.

| Dockerfile   | Pod Spec  | Purpose                            |
|--------------|-----------|------------------------------------|
| `ENTRYPOINT` | `command` | Overrides the container entrypoint |
| `CMD`        | `args`    | Overrides the default arguments    |

---

## Docker Instructions

In a Dockerfile, two instructions control what runs when a container starts.

- `ENTRYPOINT`: The executable that runs when the container starts.
- `CMD`: The default arguments passed to `ENTRYPOINT`. Used only if no arguments are provided at runtime.

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

Running the container without arguments runs the default command defined in the Dockerfile.

```bash
docker run ubuntu
```

Running the container with arguments overrides the `CMD`.

```bash
docker run ubuntu 10
```

To override the `ENTRYPOINT`, use the `--entrypoint` flag.

```bash
docker run --entrypoint=watch ubuntu 20
```

---

## Override Behavior

What runs depends on which fields are set in the Pod spec.

| `command` Set? | `args` Set? | Result                                                   |
|----------------|-------------|----------------------------------------------------------|
| No             | No          | Dockerfile `ENTRYPOINT` and `CMD` run as defined         |
| No             | Yes         | Dockerfile `ENTRYPOINT` kept, `CMD` replaced             |
| Yes            | No          | Both `ENTRYPOINT` and `CMD` ignored, only `command` runs |
| Yes            | Yes         | Both `ENTRYPOINT` and `CMD` fully overridden             |

> **Note**: Setting only `command` without `args` discards both the Dockerfile `ENTRYPOINT` and `CMD`. The original `CMD` does not remain in effect.

---

## Override Default Arguments in a Pod Manifest

Kubernetes allows replicating the Docker behavior of passing command-line arguments in a Pod manifest.

```bash
docker run ubuntu 10
```

In the Pod manifest, use the `args` field to pass the argument.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-pod
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu
      args: ["10"]
```

When the Pod starts, the `args` field overrides the default `CMD` instruction in the Dockerfile.

---

## Override the Entrypoint

In a Kubernetes Pod manifest, use the `command` field to override `ENTRYPOINT` and the `args` field to override `CMD`.

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

```bash
docker run --entrypoint=watch ubuntu 10
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-pod
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu
      command: ["watch"]
      args: ["10"]
```
- `command`: Overrides the `ENTRYPOINT` instruction.
- `args`: Overrides the `CMD` instruction.

---

## Use with `kubectl` CLI

**With `--command`:** overrides the `ENTRYPOINT`. The container ignores the original `ENTRYPOINT` and `CMD` from the Dockerfile and runs `watch 10` as the full command.

```bash
kubectl run ubuntu-pod --image=ubuntu --command -- watch 10
```
- `--command`: Signals that everything after `--` overrides the `ENTRYPOINT`.
- `--`: Separator between `kubectl` flags and the container command.
- `watch`: Passed as the new `ENTRYPOINT`.
- `10`: Passed as the argument.

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-pod
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu
      command:
        - "watch"
        - "10"
```

**Without `--command`:** overrides only `CMD`. The `ENTRYPOINT` from the Dockerfile remains, and the arguments are passed to it.

```bash
kubectl run ubuntu-pod --image=ubuntu -- watch 10
```

```dockerfile
FROM ubuntu
CMD ["sleep", "5"]
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-pod
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu
      args:
        - "watch"
        - "10"
```

---

## Generate a Manifest with `--dry-run`

**Create Pod manifest with `command`**

```bash
kubectl run ubuntu-pod --image=ubuntu --command \
  --dry-run=client -o yaml -- sleep 10 > command-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu-pod
  name: ubuntu-pod
spec:
  containers:
  - command:
    - sleep
    - "10"
    image: ubuntu
    name: ubuntu-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

**Create Pod manifest with `args`**

```bash
kubectl run ubuntu-pod --image=ubuntu \
  --dry-run=client -o yaml -- sleep 10 > args-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu-pod
  name: ubuntu-pod
spec:
  containers:
  - args:
    - sleep
    - "10"
    image: ubuntu
    name: ubuntu-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

---
