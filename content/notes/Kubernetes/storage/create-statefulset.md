---
title: Create a StatefulSet
date: 2026-03-20
weight: 6
tags: [StatefulSet]
---
# StatefulSets

A StatefulSet runs a group of Pods and maintains a sticky identity for each one. It is designed for stateful applications: workloads that require stable identity, persistent storage, and ordered operations. They are like Deployments and DaemonSets core controllers but solve a fundamentally different problem.

## How Deployments Handle Pods

Deployments are designed for **stateless applications**. When you create a Deployment with multiple Pods, Kubernetes uses a ReplicaSet to manage them. Each Pod receives a randomized name:

```
longhorn-ui-86f47785c4-6sbqw
<deployment-name>-<replicaset-id>-<random-suffix>
```

While you can attach a Persistent Volume to a Deployment for persistent storage, all Pods in that Deployment share the same volume, meaning they all access identical data.

During a rolling update, Deployments create a new ReplicaSet and gradually transition traffic from the old Pods to the new ones. This works well for stateless applications but presents challenges for stateful workloads.

---

## How StatefulSets Differ

StatefulSets do not use ReplicaSets. Instead, each Pod maintains a **sticky identity** that persists across restarts and rescheduling.

When you attach storage to a StatefulSet, each Pod creates its own Persistent Volume Claim (PVC) against a StorageClass. A StatefulSet with three replicas creates three Pods, three PVCs, and three Persistent Volumes (PVs), one per Pod:

1. `pod-nginx-0` -> `pvc` -> `pv`
2. `pod-nginx-1` -> `pvc` -> `pv`
3. `pod-nginx-2` -> `pvc` -> `pv`

StatefulSets are valuable for applications requiring one or more of the following:

- **Stable, unique network identifiers**: Pods maintain consistent DNS names.
- **Stable persistent storage**: Each Pod retains its own data across restarts.
- **Ordered, graceful deployment and scaling**: Pods are created and terminated in a predictable sequence.
- **Ordered, automated rolling updates**: Updates proceed in a controlled, sequential manner.

---

## Ordered Pod Creation and Deletion

By default, StatefulSets use the `OrderedReady` pod management policy. Pods are created sequentially in ascending order (0→1→2) and terminated in reverse order (2→1→0). Each Pod must be in Running and Ready state before the next one is created.

```
NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          110s
nginx-1   1/1     Running   0          108s
nginx-2   1/1     Running   0          106s
```

The `Parallel` policy launches or terminates all Pods simultaneously, similar to a Deployment. Use this when stable identity and storage are required, but the ordered startup is not:

```yaml
spec:
  podManagementPolicy: Parallel
```

---

## Creating a StatefulSet

Kubernetes does not provide a CLI shortcut for StatefulSets. Use a Deployment as a starting template since the specifications are nearly identical.

**Generate a Deployment manifest as a base**:
```bash
kubectl create deployment nginx-statefulset --image=nginx --replicas=3 \
  --dry-run=client --output=yaml \
  > nginx-statefulset.yaml
```

Edit the file into a StatefulSet:

1. Change `kind` to `StatefulSet`.
2. Add `serviceName: nginx-headless-svc` under `spec` (required for stable network identities).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx-app
  name: nginx-statefulset
spec:
  serviceName: nginx-headless-svc
  replicas: 3
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - image: nginx
        name: nginx-container
```

**Apply the StatefulSet**:
```bash
kubectl apply -f nginx-statefulset.yaml
```

**Verify the StatefulSet**:
```bash
kubectl get statefulsets -o wide
```
```
NAME                READY   AGE   CONTAINERS   IMAGES
nginx-statefulset   3/3     76s   nginx        nginx
```

**Verify the Pods**:
```bash
kubectl get pods
```
```
NAME                  READY   STATUS    RESTARTS   AGE
nginx-statefulset-0   1/1     Running   0          16s
nginx-statefulset-1   1/1     Running   0          14s
nginx-statefulset-2   1/1     Running   0          12s
```

Pod names follow the pattern `<statefulset-name>-<ordinal>`, starting from zero. If a Pod is deleted or rescheduled, its IP address changes, but its network name remains constant.

---

## Configuring Stable Network IDs with Headless Services

To ensure stable network identities, you need a **headless Service** that manages DNS records for the StatefulSet pods.

**Create a headless Service by setting `ClusterIP` to `None`**:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-app
  name: nginx-headless-svc
spec:
  clusterIP: None
  selector:
    app: nginx-app
  type: ClusterIP
```

**Apply the headless Service**:
```bash
kubectl apply -f nginx-headless-svc.yaml
```

**Verify the Service**:
```bash
kubectl get service nginx-headless-svc
```
```
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx-headless-svc   ClusterIP   None         <none>        <none>    9s
```

**Verify the endpoints**:
```bash
kubectl get endpoints nginx-headless-svc -o yaml
```
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: nginx-app
    service.kubernetes.io/headless: ""
  name: nginx-headless-svc
  namespace: labs
subsets:
- addresses:
  - hostname: nginx-statefulset-0
    ip: 10.0.0.215
    nodeName: k8s-worker-1
    targetRef:
      kind: Pod
      name: nginx-statefulset-0
      namespace: labs
  - hostname: nginx-statefulset-1
    ip: 10.0.0.140
    nodeName: k8s-worker-1
    targetRef:
      kind: Pod
      name: nginx-statefulset-1
      namespace: labs
  - hostname: nginx-statefulset-2
    ip: 10.0.0.181
    nodeName: k8s-worker-1
    targetRef:
      kind: Pod
      name: nginx-statefulset-2
      namespace: labs
```

You'll see references, `hostname`, and `ip` for each Pod instance in the StatefulSet.

Each Pod receives a stable DNS name following this format:

```
nginx-statefulset-0.nginx-headless-svc.labs.svc.cluster.local
nginx-statefulset-1.nginx-headless-svc.labs.svc.cluster.local
nginx-statefulset-2.nginx-headless-svc.labs.svc.cluster.local
# <hostname>.<service>.<namespace>.svc.cluster.local
```

- `.svc`: prefix subdomain.
- `.cluster.local`: default cluster DNS domain.

**Test DNS resolution from a temporary Pod**:
```bash
kubectl run curl --image=curlimages/curl -it --rm -- sh
curl nginx-statefulset-0.nginx-headless-svc.labs.svc.cluster.local
```

A successful response returns the NGINX welcome page.

---

## Rolling Update Strategies with Partitions

StatefulSets support a `partition` field under `updateStrategy`. Pods with an ordinal greater than or equal to the partition value are updated; those below it remain unchanged.

```yaml
updateStrategy:
  rollingUpdate:
    partition: 0
  type: RollingUpdate
```

The default partition is `0`, meaning all Pods are updated. Setting `partition: 2` updates only `nginx-statefulset-2` and higher.

This enables staged rollouts and canary deployments. For example, setting `partition: 2` updates only `nginx-2` and higher (with ordinals starting at zero).

### Canary Deployment

A Canary Deployment is a rollout strategy where you release a new version of the application to a small subset for testing, monitor its performance, and then gradually roll it out to the rest if everything looks good.

**Set the partition**:
```bash
kubectl edit statefulset nginx-statefulset
# set partition: 2
```

**Update the image**:
```bash
kubectl set image statefulsets nginx-statefulset nginx-container=nginx:alpine && \
  kubectl rollout status statefulset nginx-statefulset
```
```
statefulset.apps/nginx-statefulset image updated
Waiting for partitioned roll out to finish: 0 out of 1 new pods have been updated...
partitioned roll out complete: 1 new pods have been updated...
```

**Verify which Pods were updated**:
```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```
```
NAME                  IMAGE
nginx-statefulset-0   nginx
nginx-statefulset-1   nginx
nginx-statefulset-2   nginx:alpine
```

Only `nginx-statefulset-2` carries the new image. `nginx-statefulset-0` and `nginx-statefulset-1` remain on the original.

---

## Persistent Storage with StatefulSets

Using **volumeClaimTemplates**, each Pod automatically provisions its own PVC. Add a `volumeClaimTemplates` section to the StatefulSet spec:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx-app
  name: nginx-statefulset
spec:
  serviceName: nginx-headless-svc
  #...
  volumeClaimTemplates:
  - metadata:
      name: www-pvc
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: longhorn
      resources:
        requests:
          storage: 1Gi
```

Also, add the corresponding `volumeMounts` in the container spec:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx-app
  name: nginx-statefulset
spec:
  serviceName: nginx-headless-svc
  #...
  template:
    #...
    spec:
      containers:
      - image: nginx
        name: nginx-container
        volumeMounts:
        - name: www-pvc
          mountPath: /usr/share/nginx/html
```

> **Note**: Fields such as `volumeClaimTemplates` cannot be updated in place. Delete and recreate the StatefulSet to apply changes.

**Recreate the StatefulSet**:
```bash
kubectl delete statefulset nginx-statefulset
kubectl apply -f nginx-statefulset.yaml
```

**Verify PVCs**:
```bash
kubectl get pvc
```
```
NAME                          STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS
www-pvc-nginx-statefulset-0   Bound    pvc-...   1Gi        RWO            longhorn
www-pvc-nginx-statefulset-1   Bound    pvc-...   1Gi        RWO            longhorn
www-pvc-nginx-statefulset-2   Bound    pvc-...   1Gi        RWO            longhorn
```

PVC names follow the pattern `<pvc-name>-<statefulset-name>-<ordinal>`.

**Verify PVs**:
```bash
kubectl get pv
```
```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
pvc-...   1Gi        RWO            Delete           Bound    www-pvc-nginx-statefulset-0
pvc-...   1Gi        RWO            Delete           Bound    www-pvc-nginx-statefulset-1
pvc-...   1Gi        RWO            Delete           Bound    www-pvc-nginx-statefulset-2
```

The reclaim policy is set to `Delete` with dynamic provisioning.

### Testing Data Persistence

**Write a test file into a Pod**:
```bash
kubectl exec -it nginx-statefulset-0 -- sh
echo "Hello StatefulSet" > /usr/share/nginx/html/test.txt
exit
```

**Delete the Pod**:
```bash
kubectl delete pod nginx-statefulset-0
```

The StatefulSet recreates the Pod automatically. Once it is Running, verify the data survived:

```bash
kubectl exec -it nginx-statefulset-0 -- cat /usr/share/nginx/html/test.txt
```

### Data Persistence Across StatefulSet Deletion

**Delete the StatefulSet**:
```bash
kubectl delete statefulset nginx-statefulset
```

**Verify PVCs and PVs remain**:
```bash
kubectl get pvc
kubectl get pv
```

Both remain intact. Recreating the StatefulSet reattaches Pods to their original PVCs.

**Recreate the StatefulSet**:
```bash
kubectl apply -f nginx-statefulset.yaml
```

This will automatically reattach to the existing PVCs rather than creating new Pods.

**Verify the test file is still present**:
```bash
kubectl exec -it nginx-statefulset-0 -- cat /usr/share/nginx/html/test.txt
```

---

## Cleanup

```bash
kubectl delete statefulset nginx-statefulset
kubectl delete service nginx-headless-svc
for pvc in $(kubectl get pvc -o name); do kubectl delete $pvc; done
```

PVs are cleaned up automatically because the reclaim policy is set to `Delete` with dynamic provisioning.

---

## Conclusion

StatefulSets provide stable network identities, dedicated persistent storage per Pod, ordered deployment and scaling, and partition-based update strategies. Understanding how they differ from Deployments is essential for running stateful workloads in Kubernetes.

---