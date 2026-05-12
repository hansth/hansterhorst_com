---
title: Kubernetes Architecture
date: 2026-05-12
draft: false
tags: ["Kubernetes", "Architecture"]
weight: 1
---
# Kubernetes Architecture

**Kubernetes** is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications across clusters.

Kubernetes is deployed as a **cluster**, a set of servers that share workloads, improve reliability, and scale applications efficiently.

Kubernetes's **horizontal scalability** enables it to support clusters comprising thousands of nodes, distributed across multiple data centers and even regions.

---

## Cluster Architecture

A Kubernetes cluster uses a **Control Plane / Worker Node** architecture. The Control Plane acts as the brain, making cluster decisions and managing the cluster state, while the Worker Nodes run the actual application workloads.

![Cluster Architecture](/k8s-architecture.svg)

### Control Plane

The control plane is a set of components responsible for container orchestration and maintaining the cluster's desired state. It can run on a single server or across multiple servers for high availability.

### Worker Node

The Worker node executes workloads in containers managed by Pods. They are managed by the control plane and include all the services needed to run containers.

Every Kubernetes node runs three core components: the `kubelet`, a container runtime such as `containerd` or CRI-O, and `kube-proxy` — though `kube-proxy` is optional when a CNI plugin handles Service routing instead.

---

## Container Runtimes

Container runtimes are responsible for running containers. Kubernetes uses a two-layer model: a **high-level runtime** manages the overall container lifecycle, while a **low-level runtime** starts containers using Linux kernel features.

### Low-level Container Runtimes

The **low-level container runtime** creates and runs containers by interacting directly with two Linux kernel features:

- **Namespaces**: Provide process isolation, giving each container its own view of system resources — process IDs, network interfaces, mount points, and so on.
- **cgroups (Control Groups)**: Enforce resource limits to ensure that containers consume only their allocated share of CPU, memory, disk I/O, and network bandwidth.

**runc** is the default low-level container runtime implementation. It was developed by Docker and later donated to the **Open Container Initiative (OCI)**, where it became the standard OCI-compatible runtime.

Other low-level runtimes exist for specific needs:

- **crun**: A container runtime implemented in C rather than Go (like `runc`), offering potentially better performance and a smaller memory footprint
- **Kata Containers**: Provides additional isolation by running containers inside lightweight virtual machines, useful for multi-tenant environments requiring stronger security boundaries
- **gVisor**: Offers an application kernel that intercepts system calls, providing enhanced security isolation

Low-level container runtimes are installed as dependencies and used by high-level container runtimes when a container needs to be started.

### High-level Container Runtimes

**High-level container runtimes**, also called **container engines**, manage the **entire container lifecycle** by handling tasks such as pulling images from registries, configuring networking, and delegating container creation to the low-level runtime.

**`containerd`** is the standard container runtime for Kubernetes. Created by Docker, it was donated to the CNCF and is now the most widely deployed option.

**CRI-O** is another widely used high-level container runtime built specifically for Kubernetes. It implements the Kubernetes **Container Runtime Interface (CRI)** directly.

### How containerd and runc Work Together

Installing `containerd` via a package manager automatically installs `runc` as a dependency. They are separate binaries that work together to provide a complete container runtime:

- **containerd** manages the container lifecycle: image pulling, networking, storage, and coordination.
- **runc** interacts directly with the Linux kernel to set up namespaces, cgroups, and start container processes.

They communicate using the **OCI** specification. **containerd** manages the workflow and instructs **runc** to create a container. After **runc** creates the container and starts its process, **runc** exits. **containerd** then continues managing the running container via the **Container Runtime Interface (CRI)**.

### Container Runtime Interface (CRI)

The **Container Runtime Interface (CRI)** is a specification that enables the `kubelet` to communicate with container runtimes. By standardizing this interaction, Kubernetes can support multiple runtimes, such as containerd and CRI-O, without modifying the `kubelet` source code.

The `kubelet` uses the **CRI** to instruct the container runtime to pull images, create containers, start and stop them, and retrieve logs. This communication happens over **gRPC**.

![CRI](/cri.svg)

### gRPC

**gRPC** (Google Remote Procedure Call) is the communication protocol used between the `kubelet` and the container runtime over the CRI. It uses **Protocol Buffers** for efficient serialization and handles operations such as creating, starting, and stopping containers.

---

## kubelet

Kubernetes relies on the `kubelet` to run its core components. For this reason, the `kubelet` runs on both the Control Plane and the Worker Nodes, where it manages the lifecycle of the containers running on each node.

Unlike most Kubernetes components, the `kubelet` does **not** run inside a container. It is installed directly on the host operating system as a **Linux system daemon managed by `systemd`**. It communicates with the container runtime over gRPC via the CRI.

The `kubelet` consumes **Pod specifications** written in YAML manifests and ensures the described Pods are running and healthy. It watches for Pod manifests from several sources:

- **The `kube-apiserver`**: Pod assignments for standard, scheduler-managed Pods
- **`/etc/kubernetes/manifests`**: A local directory the `kubelet` watches for static Pod definitions

This directory on a **kubeadm-bootstrapped** Control Plane node contains the manifests for the core control plane components:

```
drwxrwxr-x 2 root root 4096 May 10 13:43 .
drwxrwxr-x 7 root root 4096 May  1 13:26 ..
-rw-r--r-- 1 root root    0 Feb 26 21:48 .kubelet-keep
-rw------- 1 root root 2553 May  1 13:26 etcd.yaml
-rw------- 1 root root 4263 May 10 13:43 kube-apiserver.yaml
-rw------- 1 root root 3391 May  1 13:26 kube-controller-manager.yaml
-rw------- 1 root root 1657 May  1 13:26 kube-scheduler.yaml
```

These static Pod manifests are managed directly by the **kubelet** and completely bypass the `kube-scheduler` and `kube-controller-manager`.

When the `kubelet` finds a manifest in this directory, it processes it and forwards creation requests down the container runtime stack: `kubelet → containerd → runc`.

Key responsibilities of the `kubelet`:

- **Process PodSpecs**: Receives and processes PodSpecs from the `kube-apiserver`.
- **Mount Volumes**: Mounts persistent and ephemeral storage volumes to Pods as specified.
- **Manage Secrets and ConfigMaps**: Retrieves and injects configuration data and secrets into the Pod environment.
- **Communicates with the Container Runtime**: Sends instructions to the container runtime to manage containers.
- **Report Node and Pod Status**: Continuously monitors Pod and node health and reports status updates to the `kube-apiserver`.

---

## Pods

A **Pod** is the smallest deployable unit in Kubernetes. It contains one or more containers that share the same network namespace and IP address space and specifies how to run them.

Containers are lightweight, portable, packaged applications that contain all the dependencies required to run that application.

Pods are designed to run a single instance of a workload, and the Kubernetes control plane ensures they remain running, healthy, and properly scaled — typically by managing them through higher-level resources such as Deployments.

---

## Static Pods

**Static Pods** are managed directly by the `kubelet` rather than by the Kubernetes control plane. They bypass the `kube-apiserver`, `kube-scheduler`, and `kube-controller-manager` entirely, based on manifest files in `/etc/kubernetes/manifests`.

The core control plane components run as static Pods:

- `etcd`
- `kube-apiserver`
- `kube-scheduler`
- `kube-controller-manager`

This design ensures that the control plane can start and recover on its own, without depending on higher-level abstractions that themselves depend on the control plane.

For visibility, the `kubelet` creates **mirror pods** in the API server. You can see these via `kubectl`, but you cannot modify or delete them through the Kubernetes API.

---

## Core Control Plane Components (Static Pods)

The control plane contains four main components, each running as a static Pod on kubeadm clusters.

### etcd - Source of Truth

**etcd** is a highly available, consistent, distributed key-value store. It is Kubernetes' **source of truth**, holding all cluster state: every object (Pods, Services, Deployments), all configuration data, and all current status.

All reads and writes to `etcd` go through the `kube-apiserver`. No other component accesses `etcd` directly.

In production, `etcd` should run as a cluster with an **odd number of members**, typically 3, 5, or 7, to maintain quorum. Losing `etcd` data means losing the entire cluster state, which is why regular backups are critical.

### kube-apiserver - Central Command Center

The `kube-apiserver` serves as the **single entry point** for all communication within the cluster. Every request to or from the cluster, whether it's from users, services, or internal components, goes through this API server.

It exposes a RESTful API and is responsible for:

- Authenticating and authorizing requests.
- Validating request syntax and policy compliance.
- Persisting the desired state to `etcd`.
- Returning responses to callers.

The API server is the **only** component that communicates with `etcd` directly.

### kube-scheduler — Workload Placement

The `kube-scheduler` is responsible for assigning Pods to Nodes. It runs a continuous watch loop looking for Pods that have not yet been assigned to a node, identified by an empty `nodeName` field.

For each unscheduled Pod, it evaluates:

- Available resources on each Node (CPU, memory).
- Node affinity and anti-affinity rules.
- Taints and tolerations.
- Topology spread constraints.

It scores available Nodes, selects the best match, and updates the Pod's `nodeName` via the API server. The scheduler only makes placement decisions and does not start any containers.

### kube-controller-manager - Controls the Desired State

The `kube-controller-manager` runs a collection of **controllers**, each implemented as a control loop. These loops continuously compare the actual cluster state with the desired state stored in `etcd` and make changes to reconcile any differences.

Key controllers include:

- **Deployment Controller**: Manages Deployment objects and coordinates rolling updates and rollbacks.
- **ReplicaSet Controller**: Ensures the correct number of Pod replicas is running at all times.
- **Node Controller**: Monitors node health and responds to nodes going offline.
- **Endpoint Controller**: Populates Endpoint objects, linking Services to their backing Pods.
- **Service Account Controller**: Creates default ServiceAccounts for new namespaces.

Together, these controllers provide Kubernetes with its self-healing behavior. If a Pod crashes or a node fails, the responsible controller detects it and restores the desired state.

---

## Node Components

Node components run on every node in the cluster and provide the Kubernetes runtime environment.

### kube-proxy — Network Rules (Optional)

**`kube-proxy`** is a network proxy that runs on every Node as a **DaemonSet**. It watches the API server for Service and Endpoint changes and programs the node's networking rules — using `iptables` or `ipvs` — to implement Service routing.

It enables:

- Pod-to-Pod communication across nodes.
- External-to-Pod communication via Services.

`kube-proxy` is optional when a CNI plugin such as Cilium handles Service routing instead.

### CoreDNS — Service Discovery

**CoreDNS** is a DNS server that enables Pods to discover each other by name rather than IP address. For example, a Pod can reach the `nginx` Service in the `default` namespace using the DNS name `nginx.default.svc.cluster.local`.

Unlike the static Pod control plane components, CoreDNS runs as a **Deployment** with multiple replicas for availability. It is exposed inside the cluster via a **ClusterIP Service** called `kube-dns` in the `kube-system` namespace. The `kubelet`configures this Service's IP as the DNS resolver in every Pod's `/etc/resolv.conf`, so all Pods automatically use CoreDNS for name resolution.

### Cloud Controller Manager (Optional)

The **Cloud Controller Manager (CCM)** connects Kubernetes to public cloud provider APIs such as AWS, Azure, and GCP. It enables cloud-specific functionality, including:

- Automatic provisioning of cloud load balancers for `LoadBalancer`-type Services
- Cloud-aware node lifecycle management
- Integration with cloud storage for PersistentVolumes

In self-managed or on-premises clusters, the CCM is not needed.

---

## High-Availability Architecture

In a production Kubernetes cluster, you typically run multiple Control-Plane Nodes to ensure high availability.

In an HA cluster, you need at least **three Control Plane Nodes**, always an odd number (3, 5, or 7), so that `etcd` can maintain quorum. The `etcd` instances use the **Raft consensus algorithm** to ensure all members agree on the cluster state, even if a member becomes unavailable.

For the `kube-apiserver`, a **load balancer** sits in front of all API server endpoints. Both users and Worker Nodes connect to this stable IP address rather than to individual Control Plane nodes. Because all state is stored in `etcd` and Raft guarantees consistency, any API server instance can handle any request.

---

## kubectl and User Interaction

**`kubectl`** is the standard CLI for interacting with Kubernetes. It communicates directly with the `kube-apiserver` and is the main entry point for human-driven operations, such as creating resources, inspecting cluster state, debugging Pods, and managing configuration.

---

## How It All Works Together

With those components in place, here is the full sequence of a request when you run `kubectl apply` to create a Deployment.

### Step 1: Submit a manifest

You describe the desired state in a manifest YAML file and apply it with `kubectl apply -f nginx-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
  namespace: dev
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx-pod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 2
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - image: nginx:1.25
        name: nginx-pod
        ports:
        - containerPort: 80
```

### Step 2: API server validates and persists the request

`kubectl` sends the manifest to the `kube-apiserver`. The API server authenticates the request, validates syntax and policy compliance, and rejects anything invalid. If the request passes, it persists the Deployment object to `etcd` and returns a confirmation. At this point, **no Pods are running yet**.

### Step 3: Controller manager creates a ReplicaSet and Pods

The `kube-controller-manager` runs a watch loop against the API server. When it detects the new Deployment, the **Deployment controller** creates a corresponding **ReplicaSet**. The **ReplicaSet controller** then sees that zero of the four required Pods exist and creates four Pod objects. These are persisted to `etcd` via the API server. The Pods now exist as the desired state, but are **not yet scheduled to any node**.

### Step 4: Scheduler assigns Pods to Nodes

The `kube-scheduler` watches for Pods with no `nodeName` set. It evaluates the available Nodes, scores them based on resources and constraints, and selects the best match for each Pod. It then writes each `nodeName` assignment back to the API server and persists it in `etcd`.

### Step 5: kubelet starts the containers

The `kubelet` on the selected Node watches the API server for Pods assigned to its node. When it detects new assignments, it retrieves each Pod spec, processes it, and invokes `containerd` to pull the image and launch the container. `containerd` calls `runc` to do the actual kernel-level work. Once `runc` starts the container process, it exits, and `containerd` takes over managing the running container.

### Step 6: kubelet reports status

The `kubelet` continuously streams status updates to the API server: image pulling, container starting, container running, or any errors encountered. The API server persists this status in `etcd`, keeping the recorded cluster state in sync with the actual state on the node.

### Step 7: Controller manager reconciles continuously

The controller manager never stops watching. If a Pod crashes or is deleted, the **ReplicaSet controller** detects the difference and creates a replacement. The **Deployment controller** handles higher-level concerns such as rolling updates and rollbacks. This continuous reconciliation loop is the foundation of Kubernetes' **self-healing** behavior.

---

## Conclusion

To understand Kubernetes architecture, you need to know both what each component does and how they work together.

On each node, the `kubelet` and container runtime are responsible for the low-level work of running containers.

At the cluster level, the control plane components handle scheduling, reconcile the desired state, and coordinate the system. All components communicate through the API server, and the cluster's persistent state is stored in `etcd`.

Understanding this end-to-end flow, from `kubectl apply` to a running container, is the foundation for effective day-to-day use of Kubernetes and for mastering it for the CKA exam.

---

## Definitions

**kubelet**: A `systemd`-managed daemon running on every node. Manages the Pod lifecycle by watching the API server and `/etc/kubernetes/manifests`, ensuring that the described Pods are running and healthy.

**containerd**: A high-level container runtime managing the complete container lifecycle: image pulling, storage, networking, and execution delegation to `runc`. Runs as a `systemd` service on every node.

**runc**: A low-level OCI-compatible container runtime that interacts directly with Linux kernel namespaces and cgroups to spawn container processes. Invoked by `containerd`; exits after the container process starts.

**Container Runtime Interface (CRI)**: A gRPC-based plugin interface that allows `kubelet` to communicate with any compliant container runtime (`containerd`, CRI-O) without modification to `kubelet` itself.

**gRPC**: The communication protocol used between `kubelet` and the container runtime over the CRI. Uses Protocol Buffers for efficient serialization.

**kube-apiserver**: The central API gateway for the cluster. All internal and external communication passes through it. Validates and authorizes requests and persists the accepted state to `etcd`.

**etcd**: A distributed key-value store serving as Kubernetes' source of truth. Stores all cluster state, configuration, and metadata. Uses the Raft consensus algorithm.

**kube-scheduler**: Watches for unscheduled Pods and assigns them to Nodes based on resource availability, affinity rules, taints, tolerations, and scoring. Makes placement decisions only — does not start containers.

**kube-controller-manager**: Runs multiple control loop controllers that continuously reconcile actual cluster state with desired state stored in `etcd`. Includes Deployment, ReplicaSet, Node, and Endpoint controllers.

**kube-proxy**: A network proxy running as a DaemonSet on every node. Programs networking rules using `iptables` or `ipvs`to implement Service routing. Optional when replaced by a CNI plugin such as Cilium.

**CoreDNS**: A DNS server running as a Deployment that provides service discovery. Exposed via the `kube-dns` ClusterIP Service in `kube-system`. Enables Pods to resolve Services by DNS name rather than IP address.

**Cloud Controller Manager (CCM)**: An optional component that connects Kubernetes to cloud provider APIs. It enables capabilities such as automatic load-balancer provisioning and cloud-aware node management.

**Pod**: The smallest deployable unit in Kubernetes. One or more containers sharing a network namespace, IP address, and storage volumes.

**Static Pod**: A Pod managed directly by the `kubelet` from a manifest file in `/etc/kubernetes/manifests`, bypassing the scheduler and controllers. Used to bootstrap control plane components.

**DaemonSet**: A Kubernetes resource that ensures one copy of a specific Pod runs on every node. Automatically adds Pods to new nodes as they join the cluster.

**Deployment**: A Kubernetes resource describing how an application should run, including replica count, update strategy, and Pod template. Managed by the Deployment controller.

**Service**: A Kubernetes resource that assigns a stable IP address and DNS name to a set of Pods, enabling reliable access regardless of Pod restarts or rescheduling.

**Raft**: A distributed consensus algorithm used by `etcd` to ensure that data remains consistent across all nodes in a cluster. Requires a quorum (majority) of members to be available to accept writes.

---
