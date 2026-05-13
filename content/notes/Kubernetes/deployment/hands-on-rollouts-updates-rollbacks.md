---
title: Hands-On Rollouts, Updates, and Rollbacks
date: 2026-03-16
weight: 4
tags:
- Rollouts
- Updates
- Rollbacks
---
# Hands-On: Rollouts, Updates, and Rollbacks

Practical examples for updating a Kubernetes Deployment, inspecting rollout history, and rolling back to a previous revision.

---

## Creating a Deployment and Checking Rollout Status

Create a Deployment and inspect the rollout status and history. The `kubectl create deployment` command creates a Deployment with the specified container image. Once created, a rollout is automatically triggered.

**Create the Deployment**
```bash
kubectl create deployment hands-on-deploy --image=nginx:1.16
```
```
deployment.apps/hands-on-deploy created
```

**Check the rollout status**
```bash
kubectl rollout status deployment hands-on-deploy
```
```
Waiting for deployment "hands-on-deploy" rollout to finish: 0 of 1 updated replicas are available...
deployment "hands-on-deploy" successfully rolled out
```
The `rollout status` command waits and streams updates until all replicas are available.

**Check the rollout history**
```bash
kubectl rollout history deployment hands-on-deploy
```
```
deployment.apps/hands-on-deploy
REVISION  CHANGE-CAUSE
1         <none>
```

Each time the Pod template changes, a new revision is created.

**Inspect a specific revision**
```bash
kubectl rollout history deployment hands-on-deploy --revision=1
```
```
deployment.apps/hands-on-deploy with revision #1
Pod Template:
  Labels:  app=hands-on-deploy
           pod-template-hash=c9748b64c
  Containers:
    nginx:
      Image:        nginx:1.16
      Port:         <none>
      Host Port:    <none>
      Environment:  <none>
      Mounts:       <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
```

The `--revision` flag is useful for comparing what changed between revisions, such as the container image, labels, or environment variables.

---

## Using the `--record` Flag

The `--record` flag appends the full command that triggered the revision to the `CHANGE-CAUSE` column, making it easier to trace which command caused each change.

**Update the container image with `--record`**
```bash
kubectl set image deployment hands-on-deploy nginx=nginx:1.17 --record
```
```
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/hands-on-deploy image updated
```

**Check the rollout history**
```bash
kubectl rollout history deployment hands-on-deploy
```
```
deployment.apps/hands-on-deploy
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployments hands-on-deploy nginx=nginx:1.17 --record=true
```

**Edit the Deployment interactively with `--record`**
```bash
kubectl edit deployments.apps hands-on-deploy --record
```
```
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/hands-on-deploy edited
```

This opens the Deployment manifest in the default editor. The change takes effect once the file is saved and closed. Changing the image to `nginx:latest` creates a third revision.

**Check the rollout history**
```bash
kubectl rollout history deployment hands-on-deploy
```
```
deployment.apps/hands-on-deploy
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployments hands-on-deploy nginx=nginx:1.17 --record=true
3         kubectl edit deployments.apps hands-on-deploy --record=true
```

**Inspect revision 3**
```bash
kubectl rollout history deployment hands-on-deploy --revision=3
```
```
deployment.apps/hands-on-deploy with revision #3
Pod Template:
  Labels:  app=hands-on-deploy
           pod-template-hash=5b8bd8fc78
  Annotations:
    kubernetes.io/change-cause: kubectl edit deployments.apps hands-on-deploy --record=true
  Containers:
    nginx:
      Image:        nginx:latest
      Port:         <none>
      Host Port:    <none>
      Environment:  <none>
      Mounts:       <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
```


> **Note**: The `--record` flag has been deprecated and will be removed in a future Kubernetes release. Use `kubectl annotate` with `kubernetes.io/change-cause` instead to track changes.

**Update the annotation**
```bash
kubectl annotate deployments hands-on-deploy \
  kubernetes.io/change-cause="Update container image to nginx:latest"
```

---

## Rolling Back to the Previous Revision

The `rollout undo` command rolls back the Deployment to the previous revision. The Deployment Controller scales down the Pods in the current ReplicaSet and scales up the Pods in the previous ReplicaSet.

**Undo the rollout**
```bash
kubectl rollout undo deployment hands-on-deploy
```
```
deployment.apps/hands-on-deploy rolled back
```

After the rollback, revision 2 is updated to revision 4. Revision 2 no longer appears in the history because it's now active under the new revision number.

**Check the rollout history**
```bash
kubectl rollout history deployment hands-on-deploy
```
```
deployment.apps/hands-on-deploy
REVISION  CHANGE-CAUSE
1         <none>
3         kubectl edit deployments.apps hands-on-deploy --record=true
4         kubectl set image deployments hands-on-deploy nginx=nginx:1.17 --record=true
```

**Verify the container image**
```bash
kubectl describe deployments hands-on-deploy | grep -i image:
```
```
Image: nginx:1.17
```

---

## Rolling Back to a Specific Revision

To roll back to a specific revision instead of the previous one, use the `--to-revision` flag. First, inspect the target revision to confirm which image it contains.

**Inspect revision 1**
```bash
kubectl rollout history deployment hands-on-deploy --revision=1
```
```
deployment.apps/hands-on-deploy with revision #1
Pod Template:
  Labels:  app=hands-on-deploy
           pod-template-hash=c9748b64c
  Containers:
    nginx:
      Image:        nginx:1.16
      Port:         <none>
      Host Port:    <none>
      Environment:  <none>
      Mounts:       <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
```

**Roll back to revision 1**
```bash
kubectl rollout undo deployment hands-on-deploy --to-revision=1
```
```
deployment.apps/hands-on-deploy rolled back
```

**Verify the container image**
```bash
kubectl describe deployments hands-on-deploy | grep -i image:
```
```
Image: nginx:1.16
```

**Check the rollout history**
```bash
kubectl rollout history deployment hands-on-deploy
```
```
deployment.apps/hands-on-deploy
REVISION  CHANGE-CAUSE
3         kubectl edit deployments.apps hands-on-deploy --record=true
4         kubectl set image deployments hands-on-deploy nginx=nginx:1.17 --record=true
5         <none>
```

Revision 1 is updated to revision 5, and the Deployment is now running the original `nginx:1.16` image.

---