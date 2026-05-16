---
title: Why StatefulSet
date: 2026-03-20
weight: 4
tags: [Services, Headless Service, StatefulSet]
---
# Why StatefulSet

StatefulSets are a workload API controller designed for **stateful applications**, workloads that require stable identity, persistent storage, and ordered operations. They sit alongside Deployments and DaemonSets as core controllers, but they solve a fundamentally different problem.

---

# Why StatefulSets Exist

To understand why StatefulSets are necessary, it helps to start from first principles and work back up to Kubernetes, using database replication as an example.

Imagine you have a physical server and the task is to deploy a database. You install MySQL, create a database, and it is operational. Other applications can now write data to it.

To achieve high availability, you add more servers with MySQL on each. Those new servers start with blank databases, so the data from the original server needs to be replicated to them.

### High-Level MySQL Replication

There are several replication topologies available. The most straightforward is a **single master, multi-slave** setup:

- All data goes to the master.
- Reads can be served by either the master or the other slave.

The setup procedure, step by step, is:

1. Bring up the **Master** first.
2. Deploy **Slave-1** and perform an initial clone of the Master database.
3. Enable **continuous replication** from Master to Slave-1.
4. Wait for Slave-1 to be ready, then deploy **Slave 2** and clone data from Slave-1 to avoid overloading the master's network interface.
5. Enable **continuous replication** from Master to Slave-2.
6. Configure both slaves with the **master's hostname or address** so they know where the master is.

Two requirements stand out from this procedure:

- **Ordered startup**: the Master must come up before the slaves, and Slave 1 must be ready before Slave 2 is cloned.
- **Stable, predictable identities**: slaves must be able to address the Master and each other by a name that does not change, even if a Pod is recreated.

### Why Deployments Fall Short

Deployments are designed for **stateless applications**, and when you create a Deployment with multiple Pods, Kubernetes uses a ReplicaSet to manage them. In Kubernetes, each MySQL instance (master and slaves) would be a Pod inside a Deployment. Deployments handle scaling, rolling updates, and rollbacks well, but they have two properties that break the requirements above.

**All Pods come up simultaneously.** A Deployment does not guarantee ordering. The master and both slaves could start at the same time, so there is no way to ensure the master is ready before the slaves try to replicate from it.

**Pods get random names.** When a Deployment creates or recreates a Pod, it assigns a random suffix to the name (`mysql-7d9f4b6c8-xkp2r`). If the Master Pod crashes and is recreated, it gets a new name. The slaves were configured to point to the old Pod, which no longer exists, causing replication to break.

IP addresses have the same problem: they are dynamically assigned and change when a Pod is recreated.

### How StatefulSets Solve Both Problems

StatefulSets are almost the same as Deployments (template-based Pod creation, scaling, rolling updates, rollbacks) with two key differences.

#### Ordered, Sequential Pod Creation

StatefulSets deploy Pods one at a time. Each Pod must reach a **Running and Ready** state before the next one is created. This means:

- The master (`mysql-0`) comes up first.
- Slave 1 (`mysql-1`) comes up next, only after the master is ready.
- Slave 2 (`mysql-2`) follows after slave 1 is ready.

This directly satisfies steps 1 and 4 from the replication procedure.

#### Stable, Predictable Pod Names

StatefulSets assign each Pod a **unique index** starting at 0. The Pod name is derived from the StatefulSet name and that index.

| Ordinal | Pod Name  | Role    |
|---------|-----------|---------|
| 0       | `mysql-0` | Master  |
| 1       | `mysql-1` | Slave 1 |
| 2       | `mysql-2` | Slave 2 |

These names are **sticky**. If `mysql-0` crashes and is recreated, it comes back as `mysql-0`. The slaves can always reach the master at that address, and a new slave added later (e.g., `mysql-3`) knows to clone from `mysql-2`.

This satisfies the remaining steps: slaves can be pointed at `mysql-0` for continuous replication, and each slave knows which Pod to clone from based on its own ordinal.

### Deployment vs. StatefulSet

| Requirement                 | Deployment | StatefulSet |
|-----------------------------|------------|-------------|
| Ordered Pod startup         | No         | Yes         |
| Stable Pod names            | No         | Yes         |
| Scaling and rolling updates | Yes        | Yes         |

StatefulSets are the right tool whenever workloads require **ordered deployment** and **stable network identities**, which is the case for most stateful distributed systems such as databases, message queues, and consensus-based stores.

---

## When to Use a StatefulSet

Not every application requires a StatefulSet. The decision depends entirely on the requirements of the application you are deploying. If your instances need to come up in a particular order, or if they require a stable and predictable name on the network, then a StatefulSet is the right choice. Evaluate those requirements first and proceed with a StatefulSet only if they apply.

### Creating a StatefulSet

Creating a StatefulSet is similar to creating a Deployment. You start with a manifest file that contains a Pod manifest template. The key difference is changing the `kind` from `Deployment` to `StatefulSet`.

The StatefulSet also requires a `serviceName` field specifying the name of a **Headless Service**.

```yaml
apiVersion: apps/v1 
kind: StatefulSet 
metadata: 
  name: mysql-statefulset
  labels: 
    app: mysql-statefulset 
spec:
  replicas: 3 
  selector: 
    matchLabels: 
      app: mysql-statefulset 
  serviceName: mysql-headless-svc
  template: 
    metadata: 
      name: mysql-pod
      labels: 
        app: mysql-statefulset 
    spec: 
      containers: 
        - name: mysql-container
          image: mysql 
```

### Ordered, Graceful Deployment

When applying a StatefulSet manifest, Kubernetes creates the Pods one by one in a sequential order to ensure a graceful deployment.

```bash
kubectl apply -f mysql-statefulset.yaml
```

Each Pod receives a stable, unique DNS record on the network, which any other application can use to connect to that specific Pod directly.

### Scaling Behavior

Scaling a StatefulSet also follows the ordered, graceful approach. Each new Pod comes up, becomes ready, and only then does the next Pod start. This behavior is particularly useful when scaling stateful applications, such as MySQL databases, where each new instance must clone data from the previous instance before it can serve traffic.

```bash
kubectl scale statefulset mysql --replicas=5
```

Scaling down works in the reverse order. The last Pod is removed first, followed by the second-to-last, and so on.

The same reverse-order rule applies when deleting a StatefulSet: Pods will be deleted from last to first.

```bash
kubectl delete statefulset mysql
```

### Overriding the Default Behavior with Pod Management Policy

The ordered launch behavior described above is the default, but it can be overridden. If you want the stability and unique network identity benefits of a StatefulSet without the sequential startup constraint, you can set the `podManagementPolicy` field to `Parallel`. This instructs Kubernetes to deploy all Pods simultaneously rather than one at a time.

```yaml
spec: 
  serviceName: mysql-headless-svc
  podManagementPolicy: Parallel
```

The default value of this field is `OrderedReady`, which enforces the ordered graceful behavior described above.

---

