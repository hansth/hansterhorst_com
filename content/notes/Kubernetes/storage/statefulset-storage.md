---
title: StatefulSet Storage
date: 2026-03-20
weight: 5
tags: [Storage, StatefulSet]
---
# StatefulSets

**A StatefulSet** runs a group of Pods and maintains a sticky identity for each one. It is designed for **stateful applications**: workloads that require stable identity, persistent storage, and ordered operations.

They are like Deployments and DaemonSets, core controllers, but solve a fundamentally different problem.

---

## How Deployments Handle Pods

Deployments are designed for stateless applications. When you create a Deployment with multiple Pods, Kubernetes uses a ReplicaSet to manage them. Each Pod receives a randomized name, like:

```
nginx-7d9f4b6c8-xkp2r
<deployment-name>-<replicaset-id>-<random-suffix>
```

While you can attach a Persistent Volume to a Deployment, all Pods in that Deployment share the same volume. During a rolling update, Deployments create a new ReplicaSet and gradually transition traffic from old Pods to new ones. This works well for stateless applications but not for stateful applications.

---

## How StatefulSets Differ

StatefulSets do not use ReplicaSets. Instead, each Pod maintains a **sticky identity** that persists across restarts and rescheduling.

When you attach storage to a StatefulSet, each Pod creates its own PVC. For example, a StatefulSet with three replicas creates three Pods, three PVCs, and three PVs:

- `pod-nginx-0` → `pvc` → `pv`
- `pod-nginx-1` → `pvc` → `pv`
- `pod-nginx-2` → `pvc` → `pv`

StatefulSets are valuable for applications that need the following:

- **Stable, unique network identifiers**: Pods maintain consistent DNS names.
- **Stable persistent storage**: Each Pod retains its own data across restarts.
- **Ordered, graceful deployment and scaling**: Pods are created and terminated in a predictable sequence.
- **Ordered, automated rolling updates**: Updates proceed in a controlled, sequential manner.

---

## Why StatefulSets Exist

To understand why StatefulSets are necessary, database replication is a useful example.

The most common topology is a **single master, multi-slave** setup:

- All writings go to the master.
- Reads can be served by the master or any slave.

The setup procedure, step by step:

1. Bring up the **Master** first.
2. Deploy **Slave-1** and perform an initial clone of the Master database.
3. Enable **continuous replication** from Master to Slave-1.
4. Wait for Slave-1 to be ready, then deploy **Slave-2** and clone from Slave-1 to avoid overloading the master's network interface.
5. Enable **continuous replication** from Master to Slave-2.
6. Configure both slaves with the **master's hostname or address**.

Two requirements stand out:

- **Ordered startup**: the Master must come up before the slaves, and Slave-1 must be ready before Slave-2 is cloned.
- **Stable, predictable identities**: slaves must address the Master and each other by a name that does not change, even if a Pod is recreated.

With Deployments, both requirements are broken: all Pods come up simultaneously, and Pods get random names that change when recreated.

StatefulSets solve this with two key properties:

1. **Ordered, sequential Pod creation**: each Pod must reach Running and Ready before the next one starts.
2. **Stable, predictable Pod names**: each Pod receives a unique index starting at 0.

| Ordinal | Pod Name  | Role    |
|---------|-----------|---------|
| 0       | `mysql-0` | Master  |
| 1       | `mysql-1` | Slave 1 |
| 2       | `mysql-2` | Slave 2 |

These names are sticky. If `mysql-0` crashes and is recreated, it comes back as `mysql-0`.

### Deployment vs. StatefulSet

| Requirement                 | Deployment | StatefulSet |
|-----------------------------|------------|-------------|
| Ordered Pod startup         | No         | Yes         |
| Stable Pod names            | No         | Yes         |
| Scaling and rolling updates | Yes        | Yes         |

---

## When to Use a StatefulSet

Not every application needs a StatefulSet. If your instances must start in a specific order or require stable, predictable network identities, a StatefulSet is the right choice. First evaluate the requirements, and use a StatefulSet only if you do.

---

## Creating a StatefulSet

Kubernetes does not provide a `kubectl` command to generate StatefulSets directly. Use the `kubectl create deployment` command as a starting template because the specifications are nearly identical.

**Generate a Deployment manifest as a starting point**
```bash
kubectl create deployment nginx --image=nginx --replicas=3 \
  --dry-run=client --output=yaml \
  > statefulset.yaml
```

Edit the file:

1. Remove fields with curly brackets and the `creationTimestamp` field.
2. Change `kind` to `StatefulSet`.
3. Add `serviceName: nginx-svc` under `spec` (required for stable network identities).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  serviceName: nginx-svc
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx
          name: nginx
```

**Apply the StatefulSet**
```bash
kubectl apply -f statefulset.yaml
```

**Check the StatefulSet status**
```bash
kubectl get statefulsets -o wide
```
```
NAME    READY   AGE   CONTAINERS   IMAGES
nginx   3/3     76s   nginx        nginx
```

**Check Pod status**
```bash
kubectl get pods
```
```
NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          110s
nginx-1   1/1     Running   0          108s
nginx-2   1/1     Running   0          106s
```

Pod names follow the pattern `<statefulset-name>-<ordinal>`, starting from zero. If a Pod is deleted or rescheduled, its IP address changes, but its Pod name remains constant.

---

## Ordered Pod Creation and Deletion

By default, StatefulSets use the `OrderedReady` Pod management policy. Pods are created sequentially in ascending order (0→1→2) and terminated in reverse order (2→1→0). Each Pod must be in the Running and Ready state before the next one is created.

Setting `podManagementPolicy: Parallel` launches or terminates all Pods simultaneously while still maintaining stable identities:

```yaml
spec:
  podManagementPolicy: Parallel
```

The default value is `OrderedReady`.

---

## Stable Network IDs with Headless Services

To ensure stable network identities, StatefulSets require a **Headless Service** that manages DNS records for each Pod.

**Create a Headless Service**
```bash
kubectl create service clusterip nginx-svc \
  --clusterip=None --dry-run=client \
  -o yaml > headless-svc.yaml
```
The Service name `nginx-svc` will match the `serviceName` field in the StatefulSet.

**Apply the Headless Service**
```bash
kubectl apply -f headless-svc.yaml
```

**Verify the Service**
```bash
kubectl get service nginx-svc
```
```
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx-svc   ClusterIP   None         <none>        <none>    6s
```

**Verify the endpoints**
```bash
kubectl get endpoints nginx-svc -o yaml
```
```yaml
subsets:
  - addresses:
    - hostname: nginx-0
      ip: 10.0.1.115
      nodeName: k8s-master-2
    - hostname: nginx-1
      ip: 10.0.3.254
      nodeName: k8s-worker-1
    - hostname: nginx-2
      ip: 10.0.2.253
      nodeName: k8s-master-3
```

Each Pod receives a stable DNS name following this format:

```
<hostname>.<service-name>.<namespace>.svc.cluster.local

nginx-0.nginx-svc.dev.svc.cluster.local
nginx-1.nginx-svc.dev.svc.cluster.local
nginx-2.nginx-svc.dev.svc.cluster.local
```

**Test stable DNS resolution from a temporary curl Pod**
```bash
kubectl run curl --image=curlimages/curl -it --rm -- sh
curl nginx-1.nginx-svc.dev.svc.cluster.local
```

---

## Rolling Updates with Partitions

StatefulSets support a **partition** field in the update strategy. Pods with an ordinal number greater than or equal to the partition value are updated; those with an ordinal number below it remain unchanged.

```yaml
spec:
  updateStrategy:
    rollingUpdate:
      partition: 2
    type: RollingUpdate
```

With `partition: 2`, only `nginx-2` and higher ordinals are updated. This enables canary deployments for testing a new version before rolling out to the entire StatefulSet.

**Set the partition and update the image**
```bash
kubectl edit statefulset nginx  # set partition: 2
kubectl set image statefulset/nginx nginx=nginx:alpine
```

**Verify which Pods were updated**
```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```
```
NAME      IMAGE
nginx-0   nginx
nginx-1   nginx
nginx-2   nginx:alpine
```

Only `nginx-2` shows the new image. `nginx-0` and `nginx-1` remain on the original.

---

## Persistent Storage

Using **volumeClaimTemplates**, each Pod automatically provisions its own PVC. Add a `volumeClaimTemplates` section to the StatefulSet spec and a corresponding `volumeMounts` entry in the container spec:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  serviceName: nginx
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: longhorn
        resources:
          requests:
            storage: 1Gi
```

> **Note**: Some StatefulSet fields, such as `volumeClaimTemplates`, cannot be updated in place. Delete and recreate the StatefulSet to apply changes.

**Recreate the StatefulSet**
```bash
kubectl delete statefulset nginx
kubectl apply -f statefulset.yaml
```

**Verify PVCs**
```bash
kubectl get pvc
```
```
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS
www-nginx-0   Bound    pvc-403...   1Gi        RWO            longhorn
www-nginx-1   Bound    pvc-10a...   1Gi        RWO            longhorn
www-nginx-2   Bound    pvc-187...   1Gi        RWO            longhorn
```

PVC names follow the pattern `<pvc-name>-<statefulset-name>-<ordinal>`.

### Testing Data Persistence

**Write data to a Pod**
```bash
kubectl exec -it nginx-0 -- sh
echo "Hello StatefulSet" > /usr/share/nginx/html/test.txt
exit
```

**Delete the Pod**
```bash
kubectl delete pod nginx-0
```

**Verify data persisted after the Pod is recreated**
```bash
kubectl exec -it nginx-0 -- cat /usr/share/nginx/html/test.txt
```

**Delete the entire StatefulSet and verify PVCs survive**
```bash
kubectl delete statefulset nginx
kubectl get pvc
```

PVCs and PVs remain intact after the StatefulSet is deleted. Recreating the StatefulSet reattaches Pods to their original PVCs.

---

## Cleanup

```bash
kubectl delete statefulset nginx
kubectl delete service nginx
for pvc in $(kubectl get pvc -o name); do kubectl delete $pvc; done
```

PVs are cleaned up automatically because the reclaim policy is `Delete` with dynamic provisioning.

---

## Conclusion

StatefulSets offer a reliable way to run stateful applications in Kubernetes. They provide stable network identities, dedicated persistent storage for each Pod, ordered deployment and scaling, and advanced update strategies with support for partitions.

---

