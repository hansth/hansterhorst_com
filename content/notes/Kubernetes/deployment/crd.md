---
title: Custom Resource Definitions
date: 2026-05-17
weight: 7
tags: [CRD, Deployment]
---
# Custom Resource Definitions

A Custom Resource is an extension of the Kubernetes API that may not be available in a default Kubernetes setup. Custom resources can appear and disappear in a running cluster through dynamic registration, and administrators can update them independently. Once installed, users interact with them using `kubectl`, just like built-in resources such as Pods.

---

## Resources and Controllers

A Kubernetes Resource is a declarative API with a well-defined schema structure and endpoints. Controllers implement APIs defined by Resources and run **asynchronously** after the Resources have been written to `etcd`.

### What Is a Resource?

A resource is an object stored in `etcd`, the cluster's key-value data store. When a Deployment manifest is applied, Kubernetes writes that Deployment object to `etcd`. It can be listed, described, updated, and deleted. All these operations read or write a record in `etcd`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
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
          image: nginx:latest
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl delete deployment nginx-deployment
```

### What Is a Controller?

Creating the Resource (Deployment) does more than write data to `etcd`. Three Pods appear, matching the `replicas: 3` fields, even though no one explicitly asked for Pods. The Deployment Controller creates them.

A controller is a process that runs continuously in the background. Its task is to monitor a particular resource type and reconcile the cluster's actual state against the desired state recorded in `etcd`. The Deployment Controller watches Deployment objects. When a Deployment is created, the controller creates a ReplicaSet. The ReplicaSet Controller then creates the required number of Pods. When the Deployment is deleted, the controllers clean everything up.

Kubernetes ships with built-in controllers for all native resource types: ReplicaSets, Deployments, StatefulSets, Jobs, CronJobs, Namespaces, and more. Each runs as part of the `kube-controller-manager` process on the control plane.

> **Note**: A Resource on its own is just data. A controller is what makes data actionable.

---

## Custom Resources

Kubernetes cannot respond to every workload pattern. Any domain-specific concept that benefits from being managed declaratively through the Kubernetes API can be represented as a custom resource: a database cluster, a certificate, a machine configuration, a backup policy, or anything else an organization needs to track and act on as part of its infrastructure.

Custom resources are managed the same way as built-in resources. They are created from manifest files, stored in etcd, and interacted with using `kubectl get`, `kubectl describe`, and `kubectl delete`. The difference is that the kind and its schema must first be registered with the API server using a Custom Resource Definition.

As a concrete example, a team managing flight bookings wants to track each booking as a Kubernetes object. The desired manifest:

```yaml
apiVersion: flights.com/v1
kind: FlightTicket
metadata:
  name: my-flight-ticket
spec:
  from: Mumbai
  to: London
  number: 2
```

Applying this on a fresh cluster produces an error:

```
error: no matches for kind "FlightTicket" in version "flights.com/v1"
```

Kubernetes does not know what a `FlightTicket` is. Before objects of a new kind can be created, that kind must be registered with the Kubernetes API using a **Custom Resource Definition**.

---

## Custom Resource Definition

A CRD is itself a Kubernetes object. It tells the API server to accept objects of a given kind, validate them against a schema, and store them in etcd.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: flighttickets.flights.com
spec:
  scope: Namespaced
  group: flights.com
  names:
    kind: FlightTicket
    singular: flightticket
    plural: flighttickets
    shortNames:
      - ft
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                from:
                  type: string
                to:
                  type: string
                number:
                  type: integer
                  minimum: 1
                  maximum: 10
```

### CRD Fields

| Field                    | Description                                                                                                          |
|--------------------------|----------------------------------------------------------------------------------------------------------------------|
| `metadata.name`          | Must follow `<plural>.<group>` convention. Kubernetes rejects a CRD whose name does not match.                       |
| `spec.scope`             | `Namespaced` or `Cluster`. Controls whether objects live inside a namespace or at the cluster level.                 |
| `spec.group`             | The API group used in `apiVersion`. Organises resources by domain and prevents name collisions.                      |
| `spec.names`             | Defines `kind`, `singular`, `plural`, and `shortNames` for use in manifests and `kubectl` commands.                  |
| `spec.versions`          | Supports multiple versions. Each version declares `served` and `storage` flags. Only one can be the storage version. |
| `spec.versions[].schema` | OpenAPI v3 schema defining valid fields under `spec`. Rejects invalid values before writing to etcd.                 |

### `spec.names`

| Field        | Value           | Where it is used                                   |
|--------------|-----------------|----------------------------------------------------|
| `kind`       | `FlightTicket`  | In the `kind` field of object manifests            |
| `singular`   | `flightticket`  | In `kubectl` output and error messages             |
| `plural`     | `flighttickets` | In the API path and `kubectl api-resources` output |
| `shortNames` | `ft`            | As a shorthand in any `kubectl` command            |

After the CRD is registered, the following is equivalent:

```bash
kubectl get flighttickets
kubectl get flightticket
kubectl get ft
```

### `spec.versions`

A resource can have multiple versions as it matures, following the same lifecycle as Kubernetes built-in APIs: `v1alpha1` → `v1beta1` → `v1`. Each version carries two flags:

- `served: true`: the API server accepts requests for this version.
- `storage: true`: this is the version used when writing objects to etcd.

Only one version can have `storage: true`. When introducing `v2`, set `storage: true` on `v2` and `storage: false` on `v1`, while keeping `served: true` on both during a transition period.

### `spec.versions[].schema`

The schema defines what fields are valid inside the `spec` block using OpenAPI v3. Without a schema, the API server accepts arbitrary fields under `spec`, making it impossible to catch typos or invalid values at creation time.

In this example, `from` and `to` are free-form strings. `number` must be an integer between 1 and 10 inclusive. A manifest with `number: 0` or `number: "two"` is rejected before the object is ever written to `etcd`.

---

## Registering the CRD and Creating Objects

**Apply the CRD**

```bash
kubectl apply -f flightticket-crd.yaml
```

**Verify it was registered**

```bash
kubectl get crds
kubectl api-resources | grep flight
```

**Create, inspect, and delete a FlightTicket object**

```bash
kubectl apply -f my-flight-ticket.yaml
kubectl get ft
kubectl describe flightticket my-flight-ticket
kubectl delete flightticket my-flight-ticket
```

**Test schema validation with an invalid value**

```yaml
apiVersion: flights.com/v1
kind: FlightTicket
metadata:
  name: bad-ticket
spec:
  from: Delhi
  to: Paris
  number: 99
```

```bash
kubectl apply -f bad-ticket.yaml
```

The API server rejects this because `99` exceeds the declared maximum of `10`.

---

## Why a CRD Alone Is Not Enough

A CRD solves the first part of the problem: it registers a new kind with the API server, defines its schema, and allows objects of that kind to be created, listed, described, updated, and deleted. Objects are stored in etcd.

However, a CRD alone does not make anything happen. When a custom resource manifest is applied, Kubernetes writes the object to etcd and nothing more. No process watches for it. No action is taken. The object sits in the data store until something reads it and acts on it.

This is the same principle as built-in resources: a Deployment object sitting in etcd does nothing without the Deployment Controller watching for it. Custom resources require a custom controller for the same reason.

For the `FlightTicket` example, applying a manifest writes the booking details to etcd, but no flight is booked. No process watches for new `FlightTicket` objects and no external API is called.

```
┌──────────────────────────────────────────────────┐
│               Kubernetes Cluster                 │
│                                                  │
│  kubectl apply ──► API Server ──► etcd           │
│                         │                        │
│                         │  watch events          │
│                         ▼                        │
│                  Custom Controller               │
│                         │                        │
│                         │  HTTP call             │
│                         ▼                        │
│                  bookflight.com/api  (external)  │
└──────────────────────────────────────────────────┘
```

To make something happen, a custom controller is needed.

---

## Custom Controllers

A custom controller is a process that runs in a loop, continuously watching the Kubernetes API for events on a specific resource type. When an object of that type is created, updated, or deleted, the controller responds by taking the appropriate action, whether that is provisioning infrastructure, calling an external API, updating a status field, or reconciling dependent resources.

Every custom resource needs a controller to be useful. Without one, the resource is only data.

For `FlightTicket`, the controller watches for create, update, and delete events and responds by calling the flight booking API.

### Why Go

A controller can be written in Python. Python can call the Kubernetes API server over HTTP, watch for change events, and make further API calls. Without caching, the controller sends repeated requests to the API server for the same objects on every read. Without a queue, events pile up or are processed out of order with no built-in retry mechanism.

The Kubernetes project is written in Go, and the official client library (`client-go`) ships with **shared informers** that solve both problems:

- **Caching**: the informer maintains a local in-memory cache of watched objects. Reads come from the cache rather than the API server.
- **Queuing**: events are placed on a work queue. The controller processes them one at a time, and failed items are automatically re-queued with backoff.

### Building a Controller

The Kubernetes project maintains a reference implementation called `sample-controller`. Clone it and modify `controller.go` with custom logic rather than starting from a blank file.

```bash
git clone https://github.com/kubernetes/sample-controller.git
cd sample-controller
```

The key file is `controller.go`, which contains:

- The informer setup, which defines which resource type to watch.
- Event handler functions (`onAdd`, `onUpdate`, `onDelete`) that fire when objects change.
- The `syncHandler` function, which is the reconciliation loop where business logic lives.

For a `FlightTicket` controller, `syncHandler` is where calls to the flight booking API live:

```go
// Pseudocode
func (c *Controller) syncHandler(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    ticket, err := c.flightTicketsLister.FlightTickets(namespace).Get(name)

    // Object was deleted — cancel the booking
    if errors.IsNotFound(err) {
        return cancelFlightBooking(ticket.Status.BookingRef)
    }

    // Object is new — create the booking
    bookingRef, err := bookFlight(ticket.Spec.From, ticket.Spec.To, ticket.Spec.Number)
    if err != nil {
        return err
    }

    // Write the booking reference back onto the object status
    return c.updateFlightTicketStatus(ticket, bookingRef)
}
```

**Build and run locally**

```bash
go build -o flight-controller .
./flight-controller --kubeconfig=$HOME/.kube/config
```

### Deploying the Controller in the Cluster

For production, the controller runs inside the cluster as a Deployment.

**Dockerfile**

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o flight-controller .

FROM debian:stable-slim
COPY --from=builder /app/flight-controller /usr/local/bin/flight-controller
ENTRYPOINT ["flight-controller"]
```

**Build and push**

```bash
docker build -t your-registry/flight-controller:v1 .
docker push your-registry/flight-controller:v1
```

**Deploy as a Kubernetes Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flight-controller
  namespace: flight-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flight-controller
  template:
    metadata:
      labels:
        app: flight-controller
    spec:
      serviceAccountName: flight-controller-sa
      containers:
        - name: controller
          image: your-registry/flight-controller:v1
```

> **Note**: The controller needs a ServiceAccount with RBAC permissions to read and watch `FlightTicket` objects and update their status. This is how the controller authenticates to the API server when running inside the cluster.

---

## The Operator Framework

A CRD manifest and a controller Deployment are two separate pieces. Setting up the full system on a new cluster requires applying the CRD, deploying the controller, and then creating the custom resources.

The **Operator Framework** packages the CRD and controller together into a single deployable unit called an **Operator**. Deploying the operator in one-step registers the CRD and starts the controller automatically.

```
┌────────────────────────────────────┐
│           Flight Operator          │
│ ┌──────────────┐  ┌──────────────┐ │
│ │     CRD      │  │  Controller  │ │
│ │ (FlightTicket│  │ Deployment   │ │
│ │  definition) │  │(watches and  │ │
│ └──────────────┘  │ acts on FTs) │ │
│                   └──────────────┘ │
└────────────────────────────────────┘
```

### The etcd Operator

The `etcd` Operator manages `etcd` clusters running inside Kubernetes. It creates and manages an `EtcdCluster` CRD, watches for `EtcdCluster` objects and provisions etcd Pods, exposes additional CRDs for `EtcdBackup` and `EtcdRestore`, takes backups on a schedule, and restores from a backup when a restore object is created.

This mirrors what an administrator would do: install the application, keep it running, take regular backups, and restore when something goes wrong. Kubernetes Operators encode that operational knowledge into code.

### Finding Operators

The [OperatorHub](https://operatorhub.io/) catalogue lists community and vendor-maintained operators for many popular applications, including etcd, MySQL, PostgreSQL, Prometheus, Grafana, Argo CD, and Rook-Ceph.

Installing an operator follows three steps:

1. Install the **Operator Lifecycle Manager (OLM)**, which manages the installation, upgrade, and removal of operators on the cluster.
2. Install the operator itself using OLM.
3. Create custom resource objects to provision the application.

```bash
# Install the Operator Lifecycle Manager
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.28.0/install.sh | bash -s v0.28.0

# Install the operator
kubectl create -f https://operatorhub.io/install/etcd.yaml

# Create a custom resource to provision an etcd cluster
kubectl apply -f etcd-cluster.yaml
```

---

## Reference

| Concept           | Role                                                                      | How to Create It                                                  |
|-------------------|---------------------------------------------------------------------------|-------------------------------------------------------------------|
| Resource          | An object stored in etcd, managed via the Kubernetes API                  | Built-in to Kubernetes                                            |
| Controller        | A background process that watches resources and reconciles cluster state  | Built-in for native types; custom for your own                    |
| Custom Resource   | A new object kind registered by a CRD                                     | YAML manifest applied with `kubectl`                              |
| CRD               | Registers a new kind with the API server, including schema and validation | YAML manifest applied with `kubectl`                              |
| Custom Controller | Watches for custom resource events and acts on them                       | Go program using `client-go` and the `sample-controller` scaffold |
| Operator          | Packages the CRD and controller as a single deployable unit               | Operator SDK or Kubebuilder; distributed via OperatorHub          |