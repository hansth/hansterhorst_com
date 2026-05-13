---
title: Rolling Update and Rollback
date: 2026-03-16
weight: 3
---
# Rolling Updates and Rollbacks

Creating a Deployment triggers a **rollout**, which creates a new ReplicaSet and records it as a Deployment revision. When the Pod is later upgraded with a new container image, another rollout is triggered, and a new revision is created. These revisions track the Deployment's history and allow you to roll back to a previous version if needed.

---

## Deployment Strategies

There are two strategies for updating a Deployment.

### 1. RollingUpdate

The default strategy. Pods are replaced incrementally, taking down old Pods and bringing up new ones one at a time. The application remains available throughout the update.

### 2. Recreate

All existing Pods are terminated before new ones are created. This guarantees a clean environment but introduces downtime during the transition. Not recommended for production workloads.

---

## Tracking Rollout History

**Check the rollout status**
```bash
kubectl rollout status deployment/rollingupdate-deploy
```
```
deployment "rollingupdate-deploy" successfully rolled out
```

**List the rollout history**
```bash
kubectl rollout history deployment rollingupdate-deploy
```
```
deployment.apps/rollingupdate-deploy
REVISION  CHANGE-CAUSE
1         <none>
```

The `CHANGE-CAUSE` column is empty by default. Add a description using an annotation:
```bash
kubectl annotate deployments rollingupdate-deploy \
  kubernetes.io/change-cause="Initial deployment"
```
```
REVISION  CHANGE-CAUSE
1         Initial deployment
```

---

## Updating a Deployment

You can update the Deployment in two ways, but the declarative approach is the preferred one.

### Declarative

**Modify the manifest and reapply it**
```yaml
spec:
  containers:
    - name: nginx-container
      image: nginx:mainline-alpine-perl
```

```bash
kubectl apply -f rollingupdate-deploy.yaml
```

A new rollout is triggered, and a new revision is created.

**Update the annotation to reflect the change**
```bash
kubectl annotate deployments rollingupdate-deploy \
  kubernetes.io/change-cause="Update container image to nginx:mainline-alpine-perl"
```
```
REVISION  CHANGE-CAUSE
1         Initial deployment
2         Update container image to nginx:mainline-alpine-perl
```

### Imperative

**Update the container image directly**
```bash
kubectl set image deployment/rollingupdate-deploy \
  nginx-container=nginx:stable-alpine-perl
```

**Update the annotation**
```bash
kubectl annotate deployments rollingupdate-deploy \
  kubernetes.io/change-cause="Update container image to nginx:stable-alpine-perl"
```
```
REVISION  CHANGE-CAUSE
1         Initial deployment
2         Update container image to nginx:mainline-alpine-perl
3         Update container image to nginx:stable-alpine-perl
```

> **Always modify the resource manifest file if it exists.** The imperative approach creates a mismatch between the manifest and the deployed configuration.

---

## How Upgrades Work Under the Hood

When a Deployment is updated, Kubernetes creates a new ReplicaSet with 4 replicas and scales down the old ReplicaSet to 0 replicas, which enables rollbacks if needed.

**List all ReplicaSets**
```bash
kubectl get replicasets
```
```
NAME                         DESIRED   CURRENT   READY   AGE
rollout-deploy-69c7667dbc    4         4         4       39m
rollout-deploy-6bffbb584     0         0         0       42m
rollout-deploy-6f89f7d7bc    0         0         0       41m
```

**Describe the Deployment** to see the scaling events
```bash
kubectl describe deployment rollingupdate-deploy
```

With the **RollingUpdate** strategy, the old ReplicaSet is scaled down one Pod at a time while the new ReplicaSet is scaled up one Pod at a time.

```
StrategyType:           RollingUpdate
RollingUpdateStrategy:  25% max unavailable, 25% max surge
OldReplicaSets:  rollingupdate-deploy-6bffbb584 (0/0 replicas created)
                 rollingupdate-deploy-6f89f7d7bc (0/0 replicas created)
NewReplicaSet:   rollingupdate-deploy-69c7667dbc (4/4 replicas created)
Events:
  Reason            Message
  ------            -------
  ScalingReplicaSet Scaled up rollingupdate-deploy-6b...84 to 4
  ScalingReplicaSet Scaled up rollingupdate-deploy-6f...bc to 1
  ScalingReplicaSet Scaled down rollingupdate-deploy-6b...84 to 3 from 4
  ScalingReplicaSet Scaled up rollingupdate-deploy-6f...bc to 2 from 1
  ScalingReplicaSet Scaled down rollingupdate-deploy-6b...84 to 2 from 3
  ScalingReplicaSet Scaled up rollingupdate-deploy-6f...bc to 3 from 2
  ScalingReplicaSet Scaled down rollingupdate-deploy-6b...84 to 1 from 2
  ScalingReplicaSet Scaled up rollingupdate-deploy-6f...bc to 4 from 3
  ScalingReplicaSet Scaled down rollingupdate-deploy-6b...84 to 0 from 1
```

With the **Recreate** strategy, the old ReplicaSet is scaled down to zero before the new one is scaled up.

```
StrategyType:  Recreate
OldReplicaSets:  recreate-deploy-6c7784cbd8 (0/0 replicas created)
                 recreate-deploy-7b446f8659 (0/0 replicas created)
NewReplicaSet:   recreate-deploy-6545d8fbd7 (4/4 replicas created)
Events:
  Reason            Message
  ------            -------
  ScalingReplicaSet Scaled up recreate-deploy-6c...d8 to 4
  ScalingReplicaSet Scaled down recreate-deploy-6c...d8 to 0 from 4
  ScalingReplicaSet Scaled up recreate-deploy-7b...59 to 4
  ScalingReplicaSet Scaled down recreate-deploy-7b...59 to 0 from 4
  ScalingReplicaSet Scaled up recreate-deploy-65...d7 to 4
```

---

## Rolling Back

If an update causes issues, roll back to the previous revision:

**Undo the rollout**
```bash
kubectl rollout undo deployment rollout-deploy
```

Kubernetes scales up the old ReplicaSet and scales down the current one, restoring the previous image version. The rollout history reflects the rollback as a new revision.

```bash
kubectl get replicasets
```

```
NAME                         DESIRED   CURRENT   READY   AGE
rollout-deploy-69c7667dbc    0         0         0       48m
rollout-deploy-6bffbb584     0         0         0       51m
rollout-deploy-6f89f7d7bc    4         4         4       50m
```

---

## Common `kubectl` Commands

**Check rollout status**:

```bash
kubectl rollout status deployment/<name>
```

**List rollout history**:

```bash
kubectl rollout history deployment/<name>
```

**Annotate a revision**:

```bash
kubectl annotate deployment/<name> kubernetes.io/change-cause="<message>"
```

**Update the container image**:

```bash
kubectl set image deployment/<name> <container>=<image>
```

**Roll back to the previous revision**:

```bash
kubectl rollout undo deployment/<name>
```

---
