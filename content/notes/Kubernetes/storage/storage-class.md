---
title: Storage Class
date: 2026-03-19
weight: 3
tags: [Storage, Storage Class]
---
# Storage Class

When a PVC is created from a Google Cloud persistent disk. The problem is that before this PV can be created, the disk must already exist on Google Cloud. Every time an application requires storage, first manually provision the disk on Google Cloud, then manually create a persistent volume definition file named after the disk you created.

This approach is called **static provisioning** of volumes.

---

## Dynamic Provisioning with Storage Classes

It would be far more convenient if the volume were provisioned automatically when the application requires it. That is exactly where storage classes come in.

With Storage Classes, you can define a **provisioner**, such as Google Storage, that automatically provisions storage on Google Cloud and attaches it to pods. When a claim is made, this is called **dynamic provisioning** of volumes.

You create a storage class object with the API version set to `storage.k8s.io/v1`. You specify a name and set the provisioner to `kubernetes.io/gce-pd`.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```

---

## How Storage Classes Change the Architecture

Going back to the original setup where a pod uses a PVC for storage, and the PVC is bound to a PV, we now introduce a storage class into that picture.

With a storage class in place, **you no longer need to write a PV definition file**. The PV and any associated storage are created automatically when the storage class is created. For the PVC to use the storage class you defined, you specify the storage class name in the PVC definition. That is how the PVC knows which storage class to use.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: google-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: google-storage
  resources:
    requests:
      storage: 500Mi
```

The next time a PVC is created, the storage class associated with it uses the defined provisioner to:

1. Provision a new disk with the required size on GCP.
2. Create a Persistent Volume automatically.
3. Bind the PVC to that volume.

Note that a PV is still created behind the scenes. The difference is that you no longer have to create it manually. The storage class handles that automatically.

---

## Available Provisioners and Parameters

The GCE provisioner is just one option. Many other provisioners are available, including:

- AWS EBS
- Azure File
- Azure Disk
- CephFS
- Portworx
- ScaleIO

Each provisioner supports additional parameters specific to the underlying infrastructure. For Google Persistent Disk, the notable parameters are:

| Parameter          | Options                 |
|--------------------|-------------------------|
| `type`             | `pd-standard`, `pd-ssd` |
| `replication-type` | `none`, `regional-pd`   |

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

---

## Creating Tiers of Storage

Because you can define multiple storage classes with different parameters, you can create distinct service tiers. For example:

- **Silver** — standard disk, no replication
- **Gold** — SSD drive
- **Platinum** — SSD drive with regional replication

This is precisely why the feature is called a storage class. You are defining classes of service backed by different storage characteristics. When a developer creates a PVC, they simply specify which class of storage their workload requires, and the appropriate infrastructure is provisioned automatically.

---