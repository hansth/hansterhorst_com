---
title: Pods
date: 2026-03-17
weight: 1
tags: [Pods]
---
# Pods

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers and defines how they run inside a shared Linux namespace on a worker node.

```
┌─────────────────────────────────────────────────┐
│ ┌─────────────────────────────────────────────┐ │
│ │ ┌─────────────┐ ┌─────────────────────────┐ │ │
│ │ │ ┌─────────┐ │ │ ┌─────────┐ ┌─────────┐ │ │ │
│ │ │ │Container│ │ │ │Container│ │Container│ │ │ │
│ │ │ └─────────┘ │ │ └─────────┘ └─────────┘ │ │ │
│ │ │ Pod         │ │ Pod                     │ │ │
│ │ └─────────────┘ └─────────────────────────┘ │ │
│ │ Node                                        │ │
│ └─────────────────────────────────────────────┘ │
│ Kubernetes Cluster                              │
└─────────────────────────────────────────────────┘
```

---

## The Role of Pods

A Pod represents a single instance of a running workload. When demand increases, you scale by adding more Pods, not by adding more containers within an existing Pod. Higher-level resources, such as Deployments, manage this process, ensuring Pods stay healthy, running, and correctly distributed across nodes.

---

## Multi-Container Pods

Most Pods run a single container. In some cases, a supporting container runs alongside the main application to handle tasks like logging or data processing. Both containers start and stop together, sharing the same network namespace and storage volumes. Because they share a network context, containers within the same Pod communicate over `localhost`.

---

## Deploying a Pod

**Imperative**: Create a Pod directly from the command line:

```bash
kubectl run nginx --image=nginx
```

This pulls the `nginx` image from Docker Hub by default. Check the status with:

**List all Pods**
```bash
kubectl get pods
```

The Pod will briefly show `ContainerCreating` before transitioning to `Running`.

---

## Defining a Pod with YAML

**Declarative**: Define the desired state in a manifest file and apply it to the cluster:

```yaml
apiVersion: v1
kind: Pod
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
- `apiVersion`: Kubernetes API version for this resource type
- `kind`: The resource type
- `metadata`: Identifying information: name, namespace, labels
- `spec`: The desired state: containers, volumes, and configuration

**Apply the manifest with**
```bash
kubectl apply -f pod.yaml
```

This will create the resource if it does not exist or update it if the manifest has changed.

---

## Common `kubectl` Commands

**Create a Pod**
```bash
kubectl run nginx --image=nginx
```

**List all Pods**
```bash
kubectl get pods
```

**List all Pods with node and IP information**
```bash
kubectl get pods -o wide
```

**Describe a Pod in detail**
```bash
kubectl describe pod nginx
```

This provides a full description of the Pod, including its node assignment, labels, container state, volume mounts, conditions, and recent events, as well as the main approach to debugging Pod issues.

```
Name:             nginx
Namespace:        default
Priority:         0
Service Account:  default
Node:             k8s-worker-1/192.168.3.140
Start Time:       Thu, 29 Jan 2026 15:29:59 +0100
Labels:           run=nginx
Annotations:      <none>
Status:           Running
IP:               10.0.3.190
IPs:
  IP:  10.0.3.190
Containers:
  nginx:
    Container ID:   containerd://276508e9...
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:c881927c...
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 29 Jan 2026 15:30:01 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount ...
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  kube-api-access-6qjtt:
    Type:                    Projected ...
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:... 300s
                             node.kubernetes.io/unreachable:...
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m1s   default-scheduler  Successfully assigned ...
  Normal  Pulling    5m     kubelet            Pulling image "nginx"
  Normal  Pulled     4m59s  kubelet            Successfully pulled image ...
  Normal  Created    4m59s  kubelet            Created container nginx
  Normal  Started    4m59s  kubelet            Started container nginx
```

---
