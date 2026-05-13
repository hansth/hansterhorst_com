---
title: Deployment
date: 2026-03-17
weight: 2
tags: [Deployment]
---
# Deployments

A Deployment is a Kubernetes resource that provides declarative updates to Pods and ReplicaSets. It manages a set of Pods that run an application workload.

A Deployment describes the desired state of the application: the number of replicas to run, the container image to use, and other configuration details. The Deployment Controller continuously reconciles the actual state with the desired state defined in the manifest. If a Pod crashes or a node fails, the Deployment ensures new Pods are created to replace it.

---

## Key Features

Deployments provide three key capabilities for managing Pods:

1. **Replication**: Specifies how many Pods of an application should be running. If a Pod crashes or is deleted, the Deployment automatically creates a replacement to maintain the desired replica count.
2. **Updates**: Enables you to change the desired state while keeping the application healthy. Updates roll out gradually instead of affecting all instances at once, so the application remains available during the transition.
3. **Rollbacks**: Offers a safety net when updates fail or cause problems. Deployments let you revert to a previous version, making updates safer and reducing the risk of downtime.

> Deployments manage ReplicaSets behind the scenes, offering a declarative approach to deploying and managing applications.

---

## How it Works

When you create a Deployment, it creates a ReplicaSet in the background that manages the Pods. This layered structure, Deployment → ReplicaSet → Pods, enables the **RollingUpdate** strategy.

When you update a Deployment with a new container image version, it creates a new ReplicaSet with the updated version and scales it up while scaling down the old one. It can also roll back to a previous version if something goes wrong, allowing zero-downtime deployments.

---

## Creating a Deployment

The Deployment manifest is similar to a ReplicaSet; the only difference is the `kind` type:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-app
    type: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx-container
          image: nginx
```
- `replicas`: Number of running Pods
    - `selector`: Selects Pods to manage using label matching
        - `matchLabels.app`: Must match the template labels
    - `template`: Defines the Pod spec used to create new Pods
        - `labels.app`: Referenced by the selector

**Create the resource**
```bash
kubectl create -f deployment.yaml
```

**Verify the Deployment**
```bash
kubectl get deployments
```

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           14s
```

**Verify the ReplicaSet**
```bash
kubectl get replicasets
```

```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-69bc9958f7   2         2         2       2m45s
```

**List the Pods**
```bash
kubectl get pods
```

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-69bc9958f7-b9rcb   1/1     Running   0          4m25s
nginx-deployment-69bc9958f7-v77jg   1/1     Running   0          4m25s
```

**List all resources**
```bash
kubectl get all
```

```
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-69bc9958f7-b9rcb   1/1     Running   0          4m54s
pod/nginx-deployment-69bc9958f7-v77jg   1/1     Running   0          4m54s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           4m54s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-69bc9958f7   2         2         2       4m54s
replicaset.apps/nginx-rs                      2         2         2       50m
```

---

## Deployment Strategies

There are two primary strategies for updating a Deployment:

### 1. RollingUpdate

The default strategy. Updates Pods incrementally to keep the application available, gradually terminating old Pods while bringing up new ones.

The rolling update behavior is controlled by two parameters:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

| Parameter        | Description                                     | Example with 4 Pods                                    |
|------------------|-------------------------------------------------|--------------------------------------------------------|
| `maxSurge`       | How many extra Pods can exist during a rollout  | `25%` creates 1 extra Pod before deleting old ones     |
| `maxUnavailable` | How many Pods can be unavailable simultaneously | `25%` allows 1 Pod down; the other 3 must stay running |

### 2. Recreate

Terminates all existing Pods before creating new ones. This ensures a clean environment but introduces downtime during the transition.

Useful when:

- The application cannot handle multiple versions running together.
- Resource constraints prevent running extra Pods during updates.
- The application requires exclusive access to shared resources.

The recreate behavior is controlled by changing the `strategy.type`:
```yaml
spec:
  strategy:
    type: Recreate
```


---

## Rolling Updates

Kubernetes uses **RollingUpdate** as the default strategy to update Deployments with zero downtime. Rather than shutting down all existing Pods before starting new ones, it incrementally replaces old Pods with new ones, keeping the application available throughout the update.

When you create a Deployment, Kubernetes automatically triggers a rollout and creates `revision 1`. When you update the Pod with a new container image version, a new rollout is triggered, and `revision 2` is created. This revision history lets you track changes and roll back to a previous version if necessary.

**Check the rollout status**
```bash
kubectl rollout status deployment nginx-deployment
```
```
deployment "nginx-deployment" successfully rolled out
```

**Check the rollout history**
```bash
kubectl rollout history deployment nginx-deployment
```
```
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
```

---

## Updating a Deployment

There are several ways to update a Deployment, such as changing the container image, modifying labels, or adjusting the replica count.

**Declarative**: Modify the manifest file and apply it:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-app
    type: frontend
spec:
  replicas: 3 # increase Pods
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx-container
          image: nginx:stable-alpine3.23  # update image version
```

**Reapply the Deployment**
```bash
kubectl apply -f deployment.yaml
```

This will update the Deployment with the default `RollingUpdate` strategy.

**Imperative**: Update the container image directly:
```bash
kubectl set image deployments nginx-deployment \
nginx-container=nginx:stable-alpine3.23
```

This edits the configuration directly, creating a mismatch with the desired state in the manifest file, which is the source of truth.

> **Always modify the resource manifest file if it exists.**


**Describe the Deployment**
```bash
kubectl describe deployments nginx-deployment
```

With the default `RollingUpdate` strategy, the output shows a gradual transition:

```
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sat, 31 Jan 2026 17:49:58 +0100
Labels:                 app=nginx-app
                        type=frontend
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=nginx-pod
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx-pod
  Containers:
    nginx-container:
      Image: nginx:stable-alpine3.23
OldReplicaSets:  nginx-deployment-69bc9958f7 (0/0 replicas created)
NewReplicaSet:   nginx-deployment-ff85db59d (2/2 replicas created)
Events:
  Reason             Message
  ------             -------
  ScalingReplicaSet  Scaled up replica set nginx-deployment-69bc to 2
  ScalingReplicaSet  Scaled up replica set nginx-deployment-ff8 to 1
  ScalingReplicaSet  Scaled down replica set nginx-deployment-69bc to 1 from 2
  ScalingReplicaSet  Scaled up replica set nginx-deployment-ff8 to 2 from 1
  ScalingReplicaSet  Scaled down replica set nginx-deployment-69bc to 0 from 1
```

The Deployment starts with two Pods from the old ReplicaSet (`69bc`). After applying the update, Kubernetes creates a new ReplicaSet (`ff85`), scales it up, and scales down the old one, creating a temporary overlap of three Pods until both replicas run the new version. This provides zero downtime while respecting the `maxSurge` and `maxUnavailable` constraints.

---

## Rolling Back

If you encounter issues with a new release, Kubernetes allows you to roll back to a previous deployment revision.

**Roll back the Deployment**
```bash
kubectl rollout undo deployment nginx-deployment
```

```
deployment.apps/nginx-deployment rolled back
```

**Describe the Deployment after rollback**
```
OldReplicaSets:  nginx-deployment-ff85db59d (0/0 replicas created)
NewReplicaSet:   nginx-deployment-69bc9958f7 (2/2 replicas created)
Events:
  Reason             Message
  ------             -------
  ScalingReplicaSet  Scaled up replica set nginx-deployment-69bc to 2
  ScalingReplicaSet  Scaled up replica set nginx-deployment-ff85 to 1
  ScalingReplicaSet  Scaled down replica set nginx-deployment-69bc to 1 from 2
  ScalingReplicaSet  Scaled up replica set nginx-deployment-ff85 to 2 from 1
  ScalingReplicaSet  Scaled down replica set nginx-deployment-69bc to 0 from 1
  ScalingReplicaSet  Scaled up replica set nginx-deployment-69bc to 1 from 0
  ScalingReplicaSet  Scaled down replica set nginx-deployment-ff85 to 1 from 2
  ScalingReplicaSet  Scaled up replica set nginx-deployment-69bc to 2 from 1
  ScalingReplicaSet  Scaled down replica set nginx-deployment-ff85 to 0 from 1
```

Kubernetes scales up the old ReplicaSet (`69bc`) from 0 and scales down the current ReplicaSet (`ff85`), creating a temporary overlap of three Pods until all replicas run the previous version again. The rollback uses the same rolling update mechanism in reverse, making it fast since the old ReplicaSet's Pod template already exists.

**Check the rollout history after rollback**
```bash
kubectl rollout history deployment nginx-deployment
```

```
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

---

## Common `kubectl` Commands

**Create a Deployment**
```bash
kubectl create -f deployment.yaml
```

**List all Deployments**
```bash
kubectl get deployments
```

**Apply changes to a Deployment**
```bash
kubectl apply -f deployment.yaml
```

**Update the container image**
```bash
kubectl set image deployments nginx-deployment \
nginx-container=nginx:stable-alpine3.23
```

**Check rollout status**
```bash
kubectl rollout status deployments nginx-deployment
```

**Roll back to the previous revision**
```bash
kubectl rollout undo deployments nginx-deployment
```

## Conclusion

The difference between a ReplicaSet and a Deployment may not be obvious at first. A Deployment is essentially a higher-level Kubernetes resource that manages and wraps a ReplicaSet. The real benefits appear when you use features such as rolling updates, rollbacks, and pause-and-resume, all enabled by Deployments.

---