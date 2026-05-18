---
title: Multi-Container Pods
date: 2026-03-16
weight: 2
tags: [Pods, Containers]
---
# Multi-Container Pods

A Multi-Container Pod is a single Pod that runs two or more tightly coupled containers that need to share resources. While a single-container Pod is the most common approach, multi-container Pods allow related processes to work together as a single unit.

---

## Why Multi-Container Pods Exist

Multi-container Pods share the same lifecycle, network, and storage, and are created and destroyed when the Pod is terminated. They communicate with each other via `localhost` and access shared volumes, eliminating the need for volume sharing or Services.

---

## Creating a Multi-Container Pod

To create a multi-container Pod, add the container specifications in the `containers` section of the Pod manifest file. The `containers` field is an array to support multiple containers within a single Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: web-app
      image: nginx
    - name: log-app
      image: busybox
```

---

## Design Patterns

There are different design patterns for multi-container Pods in Kubernetes. Understanding each pattern, when to use it, and the differences between them is essential for designing effective workloads.

### Co-Located Containers

The basic form of multi-container Pods. Both containers are defined in the `containers` array and run the entire lifecycle of the Pod. There is no guaranteed startup order between them. Use this pattern when two services rely on each other and the startup order does not matter.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: co-container-pod
spec:
  containers:
    - name: main-app
      image: nginx
    - name: helper-app
      image: busybox
```

### Init Containers

Use this pattern when initialization steps must be complete before the main application starts. An init container runs to completion, then exits, and only then does the main container start. These init containers are defined in a separate `initContainers` block.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-pod
spec:
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
    - name: main-container
      image: main-image
```

Multiple init containers can be defined and run sequentially. Each must be completed before the next begins, and all must be completed before the main container starts.

For example, the first init container starts and waits until the database is initialized, and then the second init container starts to check when the external API becomes available. After all init containers are complete, the main application starts and continues to run until the Pod lifecycle ends.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-init-container-pod
spec:
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
    - name: init-api
      image: busybox
      command: ['sh', '-c', 'until nc -z api 8080; do sleep 2; done']
  containers:
    - name: main-container
      image: main-image
```

The index (`0, 1, 2,...`) of the initContainers list determines the order in which the containers are started.

### Sidecar Containers

A sidecar is an init container that never exits. It starts before the main container and runs alongside it for the entire Pod lifecycle and stops after the main container ends. This is useful for logging agents that need to capture both startup and termination output.

The sidecar is declared in `initContainers` with `restartPolicy: Always`, which tells Kubernetes to keep it running rather than treating it as a one-shot init step.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  initContainers:
    - name: logging-container
      image: busybox
      restartPolicy: Always
  containers:
    - name: main-container
      image: main-image
```


---

## When to Use Which Pattern

The init container pattern is straightforward: the init container starts, finishes, and then the main application container runs.

Co-located and sidecar containers can be more confusing because, in both patterns, all containers run for the duration of the Pod's lifecycle.

The key difference is the startup order. With co-located containers, you cannot control which container starts first. All containers are defined in the same `containers` array, with no distinction between them, so they start together with no guarantees about their order. This works well when the startup order does not matter.

The sidecar container pattern, by contrast, is typically used when you need containers to run together while still controlling their startup sequence. This lets you ensure the sidecar is ready before or alongside the main application, while both continue to run throughout the Pod lifecycle.

|               | Init Container                           | Co-Located            | Sidecar                                  |
|---------------|------------------------------------------|-----------------------|------------------------------------------|
| Declared in   | `initContainers`                         | `containers`          | `initContainers`                         |
| Startup order | Init containers start first              | No guarantee          | Sidecar starts first                     |
| Use when      | Initialization steps must complete first | Order does not matter | Supporting container must be ready first |

---

## Restart Behavior

Kubernetes treats the Pod as a single unit of execution. If any main container fails and the Pod's `restartPolicy` is `Always` or `OnFailure`, all containers in the Pod are restarted together. Kubernetes does not restart individual containers within a Pod.

This applies only to main containers. Init containers run to completion before main containers start and are not restarted individually.

---