---
title: Headless Service with StatefulSet
date: 2026-03-20
weight: 7
tags: [Services, Headless Service, StatefulSet]
---
# Headless Service with StatefulSet

Kubernetes Services provide a stable endpoint for exposing applications running on a set of Pods. While Pods are ephemeral and can be rescheduled with different IP addresses, Services provide a stable endpoint that routes traffic to the appropriate Pods.

A **Headless Service** functions like a standard Service, allowing a client to connect directly to any preferred Pod, but with one key difference: it has no cluster IP and performs no load balancing. Instead, it creates a DNS record for each Pod using the Pod name as the subdomain.

---

## Why StatefulSets Need a Headless Service

When using a StatefulSet, Pods are deployed sequentially, each with a unique name such as `mysql-0`, `mysql-1`, and `mysql-2`. In a master-slave topology, writes must go to the master Pod only, while reads can come from any slave Pod.

A standard Service is not enough for this:

- **Using the Pod's IP address** is unreliable because IP addresses are dynamic and change when a Pod is recreated.
- **Using the Pod's DNS address** is equally unreliable because Pod DNS records are derived from IP addresses and change with them.
- **Using a standard Service** load-balances across all Pods, which is not suitable when writes must target a specific Pod.

A Headless Service solves this by providing a stable DNS record per Pod.

---

## DNS Records with a Headless Service

When you create a Headless Service named `mysql-headless-svc`, each Pod gets a stable DNS record:

```
mysql-0.mysql-headless-svc.default.svc.cluster.local
mysql-1.mysql-headless-svc.default.svc.cluster.local
mysql-2.mysql-headless-svc.default.svc.cluster.local

<pod>.<headless-service>.<namespace>.svc.cluster.local
```

An application can now point directly to the master Pod using:

```
mysql-0.mysql-headless-svc.default.svc.cluster.local
```

This DNS entry always resolves to `mysql-0`, regardless of restarts or rescheduling.

---

## Creating a Headless Service

A Headless Service manifest is identical to a standard Service manifest, with one distinguishing field:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless-svc
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
```
- `clusterIP: None`: Makes the ClusterIP Service headless. No virtual IP is assigned, and no load balancing is performed.

---

## DNS Record Requirements for Pods

Creating the Headless Service alone is not enough. DNS records for individual Pods are only created when two conditions are met in the Pod manifest:

1. The `subdomain` field must be set to the same name as the Headless Service.
2. The `hostname` field must be set to give each Pod a unique identity.

Both fields are required:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  hostname: mysql-0
  subdomain: mysql-headless-svc
  containers:
    - name: mysql-container
      image: mysql
```

With a Deployment, all Pods receive the same `hostname` and `subdomain` values because the Deployment replicates the Pod specification identically. As a result, multiple Pods share the same DNS name, which prevents individual addressing.

---

## How StatefulSets Handle This Automatically

When Pods are created as part of a StatefulSet, you do not need to manually specify `hostname` or `subdomain`. The StatefulSet automatically assigns each Pod the correct hostname based on its ordinal and the correct subdomain based on the Headless Service name.

Declare the Service name in the StatefulSet using the `serviceName` field:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-sfs
spec:
  serviceName: "mysql-headless-svc"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql-container
          image: mysql
```

The StatefulSet uses the `serviceName` value to set the `subdomain` on each Pod it creates. Each Pod then automatically receives its own unique DNS A record, enabling stable individual addressing across the entire StatefulSet. This is one of the key differences between a Deployment and a StatefulSet.

---
