---
title: Persistent Storage
date: 2026-03-19
weight: 2
tags: [Storage, PV, PVC]
---
# Persistent Storage

**Kubernetes persistent storage** allows data to survive beyond the lifecycle of individual Pods, containers, or Nodes. By default, Pods are ephemeral, meaning all saved data is lost if a Pod crashes or restarts. Kubernetes abstracts the underlying storage infrastructure through a decoupled architecture that separates developers' needs from cluster administrators' management tasks.

Persistent storage operates on three levels:

| Level | Resource                     | Responsibility                                                                                                                 |
|-------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| 1     | `StorageClass`               | Defines the type and characteristics of storage. Administrators describe the classes of storage they offer.                    |
| 2     | `PersistentVolume`(PV)       | A piece of storage provisioned by an administrator or dynamically via a StorageClass. Persists beyond Pod lifecycles.          |
| 3     | `PersistentVolumeClaim`(PVC) | A request for storage by a user. Specifies the amount and characteristics needed. Once bound to a PV, it can be used by a Pod. |

When a PVC is created, Kubernetes finds or creates a PV that satisfies the claim's requirements. The PV is then referenced by Pods that need access to the storage.

### Access Modes

Access modes define how a volume can be mounted across nodes. Available modes depend on the underlying storage backend.

| Mode             | Abbreviation | Description                                   |
|------------------|--------------|-----------------------------------------------|
| ReadWriteOnce    | `RWO`        | Read-write by a single node                   |
| ReadOnlyMany     | `ROX`        | Read-only by multiple nodes                   |
| ReadWriteMany    | `RWX`        | Read-write by multiple nodes                  |
| ReadWriteOncePod | `RWOP`       | Read-write by a single Pod (Kubernetes 1.27+) |

Block storage backends such as AWS EBS or Longhorn typically support only RWO. Network filesystems such as NFS or CephFS can support RWX. The access mode describes what the storage supports and is not actively enforced at the I/O level by Kubernetes.

### Provisioning Methods

There are two approaches to creating persistent volumes:

- **Manual provisioning**: Both the PV and PVC are manually created.
- **Dynamic provisioning**: Only the PVC is created. The StorageClass automatically creates the PV.

### Reclaim Policies

Reclaim policies determine what happens to storage after a claim is released:

| Policy    | Behavior                                                                                      |
|-----------|-----------------------------------------------------------------------------------------------|
| `Retain`  | Data persists until the volume is explicitly deleted. Default for manually created PVs.       |
| `Delete`  | The associated storage asset is permanently deleted. Default for dynamically provisioned PVs. |
| `Recycle` | Data in the volume is scrubbed before it is made available to other claims. (Deprecated).     |

### Volume Binding Modes

Volume binding modes control when a PV is provisioned and bound to a PVC:

- **Immediate**: The PV is provisioned and bound as soon as the PVC is created, regardless of whether a Pod uses it. This can cause scheduling problems with topology-constrained storage.
- **WaitForFirstConsumer**: Provisioning and binding are delayed until a Pod that uses the PVC is scheduled. The scheduler then considers storage topology when placing the Pod. This is the recommended mode for topology-aware storage.

---

## Persistent Volumes

A **Persistent Volume (PV)** is a cluster-wide pool of storage volumes configured by an administrator for use by applications running in the cluster. Users can request storage from this pool using **Persistent Volume Claims (PVCs)**.

### Creating a Persistent Volume

A PersistentVolume manifest uses the standard Kubernetes resource structure. In the `spec` section you define three key fields: `accessModes`, `capacity`, and the volume type.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-volume
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /data
```

- `metadata.name`: Assigns a cluster-wide identifier to the volume.
- `accessModes`: Defines how the volume can be mounted across nodes.
- `capacity.storage`: Reserves the allocated storage for this volume.
- `hostPath.path`: Points to a directory on the node's local filesystem.

> **Note**: `hostPath` is not suitable for production environments. Replace it with a supported storage solution such as AWS Elastic Block Store.

**Create the PersistentVolume**:

```bash
kubectl apply -f pv-volume.yaml
```

**List PersistentVolumes**:

```bash
kubectl get persistentvolumes
```

```
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
pv-volume   1Gi        RWO            Retain           Available
```

---

## Persistent Volume Claims

A **PersistentVolumeClaim (PVC)** is a user's request for storage. Pods consume node resources; PVCs consume PV resources. Claims can request a specific size and access mode.

### How Binding Works

Once a PVC is created, Kubernetes binds it to a PV based on the request and properties set on the volume.

During binding, Kubernetes looks for a PersistentVolume (PV) with sufficient capacity and matching properties, such as access modes, volume modes, and storage class. A smaller PersistentVolumeClaim (PVC) can be bound to a larger PV if all other criteria are met and no better match is available. There is a one-to-one relationship between PVCs and PVs, so no other claim can use the remaining capacity of that volume.

If no PVs are available, the PVC remains in `Pending` status until a new PV becomes available. Once a matching PV is available, the PVC is automatically bound to it.

### Creating a Persistent Volume Claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

- `metadata.name`: Assigns a cluster-wide identifier to the claim.
- `accessModes`: Defines how the volume can be mounted across nodes.
- `resources.requests.storage`: The amount of storage requested.

**Apply the manifest**:

```bash
kubectl apply -f pvc-claim.yaml
```

**Verify the PVC**:

```bash
kubectl get persistentvolumeclaim pvc-claim
```

```
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
pvc-claim   Bound    pvc-70...  500Mi      RWO            longhorn
```

When Kubernetes finds the created PV and checks that the access modes are compatible, it binds the PVC to that PV, even though the PVC requests 500Mi and the PV offers 1Gi, because no other PVs are available.

---

## Using a PVC in a Pod

### Use the PVC in a Pod

Reference the PVC in a Pod manifest by adding a `persistentVolumeClaim` entry under `volumes`, then mount it into the container via `volumeMounts`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
      volumeMounts:
        - mountPath: "/var/www/html"
          name: pv-volume
  volumes:
    - name: pv-volume
      persistentVolumeClaim:
        claimName: pvc-claim
```

The `volume.name` field and `volumeMounts.name` must match. The `claimName` must match the PVC's `metadata.name`.

### Use the PVC in a Deployment

The same approach applies to a Deployment. Add the `volumes` and `volumeMounts` fields inside `spec.template.spec`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-deployment
    spec:
      containers:
        - name: nginx-container
          image: nginx
          volumeMounts:
            - mountPath: "/var/www/html"
              name: pv-volume
      volumes:
        - name: pv-volume
          persistentVolumeClaim:
            claimName: pvc-claim
```

> **Note**: `ReadWriteOnce` allows the volume to be mounted by a single node. If you scale the Deployment beyond one replica and the Pods land on different nodes, only one node will hold the volume and Pods on other nodes will fail to start. For multi-replica Deployments, use `ReadWriteMany` with a StorageClass that supports it. Longhorn supports `ReadWriteMany` via its NFS-based provisioner.

### Verify

After applying the manifests, confirm the volume is mounted inside the container:

```bash
kubectl exec nginx-deployment-df8f67cdf-z27l9 -- df -h /var/www/html
```

---

## Deleting a PVC

```bash
kubectl delete persistentvolumeclaim pvc-claim
```

When a claim is deleted, the `persistentVolumeReclaimPolicy` field controls what happens to the underlying PV:

- `Retain` **(default)**: The PV remains until manually deleted by the administrator. It is not available for reuse by other claims.
- `Delete`: The PV is deleted as soon as the claim is deleted.
- `Recycle` **(deprecated)**: Data in the volume is scrubbed before it is made available to other claims.

### Why Recycle Is Deprecated

Kubernetes deprecated the `Recycle` policy because it did not guarantee secure data erasure and only worked for certain in-tree volume plugins. Kubernetes replaced it with dynamic provisioning via StorageClasses and CSI drivers, which handle cleanup correctly at the provider level.

---

## PV vs. PVC

| Concept                   | Detail                                            |
|---------------------------|---------------------------------------------------|
| PVC to PV relationship    | One-to-one                                        |
| Binding basis             | Capacity, access mode, volume mode, storage class |
| Pending state             | Remains until a matching volume becomes available |
| Default reclaim policy    | `Retain`                                          |
| Deprecated reclaim policy | `Recycle`                                         |
| Modern approach           | Dynamic provisioning via StorageClass and CSI     |

---

## StorageClass

A **StorageClass** lets administrators describe the types of storage they offer. Each class can have different quality-of-service levels, backup policies, or other administrator-defined policies. Kubernetes does not prescribe what classes should represent.

Without a StorageClass, administrators must manually provision PVs before users can claim storage. A StorageClass removes this requirement by enabling **dynamic provisioning**: when a PVC references a StorageClass, Kubernetes automatically provisions a PV and binds the claim.

### Creating a StorageClass

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-rwx
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "4"
  staleReplicaTimeout: "30"
  dataLocality: "disabled"
  migratable: "false"
  dataEngine: "v1"
```

- `provisioner`: The CSI driver responsible for provisioning volumes. This example uses the Longhorn CSI driver.
- `allowVolumeExpansion`: Allows the volume to be resized after creation.
- `parameters`: Driver-specific configuration passed to the Longhorn provisioner.

### Using a StorageClass in a PVC

Specify the StorageClass name in the PVC. Kubernetes uses the associated provisioner to provision a new disk, create the PV, and bind the claim automatically:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-rwx
  resources:
    requests:
      storage: 500Mi
```

> **Note**: The PV is still created behind the scenes. The StorageClass handles provisioning automatically so you no longer need to define a PV manifest.

---
---