---
title: ReplicaSet
date: 2026-03-17
weight: 1
tags: [ReplicaSets, ReplicationController]
---
# ReplicaSets

A ReplicaSet is a mechanism to control the desired number of Pods that are always running in the cluster. If a Pod crashes, the ReplicaSet automatically starts a new one to maintain the desired state.

---

## ReplicaSet vs. ReplicationController

There are two similar resources: **ReplicationController** and **ReplicaSet**. Both serve the same purpose but are different.

The ReplicationController is an older technology that uses equality-based label selectors and is replaced by the more advanced ReplicaSet. ReplicaSets support set-based label selectors, which provide more flexible matching capabilities.

The `selector` field is optional in a ReplicationController but required in a ReplicaSet. The selector enables a ReplicaSet to manage existing Pods that match the specified labels, even if those Pods were not created by the ReplicaSet itself.

---

## Creating a ReplicationController

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
  labels:
    app: nginx-app
    type: frontend
spec:
  replicas: 3          # Number of running Pods
  selector:            # Optional
    app: nginx-app
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-app
        type: frontend
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

**Create the resource**
```bash
kubectl create -f replication-controller.yaml
```

**Verify the resource**
```bash
kubectl get replicationcontrollers
```

```
NAME       DESIRED   CURRENT   READY   AGE
nginx-rc   3         3         3       23s
```

**Verify the Pods**
```bash
kubectl get pods
```

```
NAME             READY   STATUS    RESTARTS   AGE
nginx-rc-8t2jx   1/1     Running   0          59s
nginx-rc-lxwtb   1/1     Running   0          59s
nginx-rc-spg2n   1/1     Running   0          59s
```

**Delete the resource**
```bash
kubectl delete -f replication-controller.yaml
```

---

## Creating a ReplicaSet

A ReplicaSet differs from a ReplicationController in two ways:

1. **API version and kind**: `apiVersion` is `apps/v1` and `kind` is `ReplicaSet`.
2. **Selector requirement**: A ReplicaSet requires a `selector` block to identify which Pods to manage. It also enables the adoption of existing Pods that match the specified labels, even if they were not created by the ReplicaSet itself.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx-app
    type: frontend
spec:
  replicas: 3         # Number of running Pods
  selector:           # Required
    matchLabels:
      type: frontend  # Must match template labels
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-app
        type: frontend
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

**Apply the resource**
```bash
kubectl apply -f replica-set.yaml
```

**Verify the resource**
```bash
kubectl get replicasets
```

```
NAME       DESIRED   CURRENT   READY   AGE
nginx-rs   3         3         3       25s
```

**Verify the Pods**
```bash
kubectl get pods
```

```
NAME             READY   STATUS    RESTARTS   AGE
nginx-rs-58hvl   1/1     Running   0          51s
nginx-rs-drhl8   1/1     Running   0          51s
nginx-rs-f7dv7   1/1     Running   0          51s
```

**Delete the resource**
```bash
kubectl delete -f replica-set.yaml
```

---

## Labels and Selectors

Labels are key-value pairs attached to a Kubernetes resource. Selectors tell the ReplicationController or ReplicaSet which Pods to manage by matching those labels.

_Equality-based_ labels allow filtering by label keys and values. Matching objects must satisfy all specified label constraints, though they may also have additional labels. Three kinds of operators are supported `=`,`==`, and `!=`.

_Set-based_ labels allow filtering keys according to a set of values. Three kinds of operators are supported: `in`, `notin`, and `exists` (only the key identifier).

A ReplicaSet with the selector `app: nginx-pod` manages all Pods labeled `app: nginx-pod`. If a Pod with that label is deleted, the ReplicaSet creates a new one. If you manually create a Pod with the same label, the ReplicaSet counts it toward the desired replica count.

**Describe the ReplicaSet**
```bash
kubectl describe rs nginx-rs
```

```
Name:         nginx-rs
Namespace:    default
Selector:     app=nginx-pod       # matches Pod label
Labels:       app=nginx-app
              type=frontend
Annotations:  <none>
Replicas:     6 current / 6 desired
Pods Status:  6 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx-pod          # matches selector
  Containers:
   nginx-container:
    Image: nginx
Events:
  # ...
```

The `Selector` and `Pod Template Labels` fields must match for the ReplicaSet to manage its Pods correctly.

---

## Scaling a ReplicaSet

There are multiple ways to scale a replica set. The declarative or the imperative approach, with two options.


**Declarative**: Update the `replicas` field in the manifest and apply it:

```yaml
spec:
  replicas: 6
```

```bash
kubectl apply -f replica-set.yaml
```

**Imperative**: Scale directly using the command line:

```bash
kubectl scale --replicas=6 -f replica-set.yaml
```

Or reference the ReplicaSet by name:

```bash
kubectl scale --replicas=6 replicaset nginx-rs
```

---

## Common `kubectl` Commands

**Create a ReplicaSet**:

```bash
kubectl create -f replica-set.yaml
```

**List all ReplicaSets**:

```bash
kubectl get replicasets
```

**List Pods managed by a ReplicaSet**:

```bash
kubectl get pods
```

**Describe a ReplicaSet**:

```bash
kubectl describe rs nginx-rs
```

**Update a ReplicaSet using a file**:

```bash
kubectl replace -f replica-set.yaml
```

**Scale a ReplicaSet**:

```bash
kubectl scale --replicas=6 replicaset nginx-rs
```

**Delete a ReplicaSet**:

```bash
kubectl delete replicaset nginx-rs
```

## Conclusion

In Kubernetes, you generally don’t work with ReplicaSets directly. In production, Deployments are the preferred way to manage Pods because they create a ReplicaSet and add features such as rolling updates, rollbacks, and revision history. When a Deployment creates a ReplicaSet, you should not modify that ReplicaSet manually, because the Deployment controller handles it and continually reconciles its desired state.

ReplicationController is now considered a legacy resource and is no longer actively developed. ReplicaSets enhance label selection with set-based selectors, while the ReplicationController only supports equality-based label selectors.

---