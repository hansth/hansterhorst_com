---
title: Ingress
date: 2026-03-18
weight: 2
tags: [Ingress]
---
# Ingress

In Kubernetes, **Ingress** is an API object that manages external access to Services via HTTP and HTTPS. It acts as a router that directs incoming traffic to the correct Services based on rules you define.

```
                       ┌────────────────────────────────────────────┐
                       │ Cluster                             ┌─────┐│
                       │                                 ┌──▶│ Pod ││
           Ingress     │   ┌──────────┐    ┌────────┐    │   └─────┘│
Client ────────────────┼──▶│ Ingress  │───▶│Service │────┤          │
          Controller   │   │ Resource │    └────────┘    │   ┌─────┐│
         (Deployment)  │   └──────────┘                  └──▶│ Pod ││
                       │                                     └─────┘│
                       └────────────────────────────────────────────┘
```

> **Note**: The client connects to the **Ingress Controller**. The **Ingress Resource** is a manifest that the controller reads to decide how to route traffic.

Ingress provides URL-based routing, load balancing, SSL/TLS termination, and virtual hosting. It is implemented by an **Ingress Controller**, which typically uses a load balancer or edge router. Ingress does not expose arbitrary ports or protocols. To expose non-HTTP/HTTPS services, use a Service of type `NodePort` or `LoadBalancer`.

Think of Ingress as a **Layer 7 load balancer** built into the Kubernetes cluster, configured with native Kubernetes primitives like other objects.

---

## How Ingress Works

Ingress has two components:

- **Ingress Controller**: The deployed solution that handles traffic (e.g., nginx, HAProxy, Traefik).
- **Ingress Resource**: A Kubernetes manifest that specifies the routing rules the controller reads and applies.

A Kubernetes cluster does not come with an Ingress Controller by default. You must deploy one. Creating Ingress Resources without a controller has no effect.

---

## Ingress Controllers

Several Ingress Controller solutions are available, including GCE, nginx, Contour, HAProxy, Traefik, and Istio. GCE and nginx are currently maintained by the Kubernetes project. This article uses nginx as the example.

An Ingress Controller is more than just a nginx server. It includes logic to watch the cluster for new or updated Ingress Resources and reconfigure nginx accordingly.

---

## Deploying the nginx Ingress Controller

The controller is deployed as a standard Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image: registry.k8s.io/ingress-nginx/controller:<version>
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 80
            - containerPort: 443
```

- `args: [/nginx-ingress-controller]`: Overrides the container's default `CMD` to run the Ingress Controller binary, which watches the API server for Ingress Resources and updates the nginx configuration accordingly.
- `args: [--configmap=$(POD_NAMESPACE)/nginx-configuration]`: Tells the controller which ConfigMap to use for nginx configuration. `$(POD_NAMESPACE)` is resolved at runtime from the environment variable defined below.
- `env`: Provides the Pod's name and namespace, used by the controller to identify itself, update Ingress status, get leader-election leases, and access ConfigMaps or Secrets from the correct namespace.
- `ports`: Exposes ports `80` (HTTP) and `443` (HTTPS), on which nginx listens for traffic forwarded by a Service.

### Supporting Resources

The Deployment alone is not enough. The following resources are also required:

**A NodePort Service** to expose the controller for external traffic:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
  selector:
    name: nginx-ingress
```

**A ConfigMap** to hold nginx configuration data:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
```

**A ServiceAccount** with the correct Roles and RoleBindings, so the controller can monitor the cluster for Ingress Resources and configure nginx accordingly:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

With these four resources in place, the Ingress Controller is operational.

---

## Ingress Resources

An Ingress Resource is a manifest that defines routing rules for the Ingress Controller. There are three common routing scenarios:

- **Default backend**: Routes all traffic to a single Service.
- **Path-based routing**: Directs traffic based on the URL path.
- **Host-based routing**: Routes traffic based on the DNS hostname.

### Default Backend

The simplest case routes all incoming traffic to a single Service on port `80`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: test-service
      port:
        number: 80
```

**Apply the Ingress Resource**:

```bash
kubectl create -f ingress.yaml
```

**Verify**:

```bash
kubectl get ingress
```

```
NAME           HOSTS   ADDRESS   PORTS   AGE
test-ingress   *       <none>    80      2m
```

---

### Path-Based Routing

To route traffic based on URL paths, define multiple path rules under a single rule:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wear-watch-path-ingress
spec:
  rules:
    - http:
        paths:
          - path: "/wear"
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80
          - path: "/watch"
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 80
```

**Apply the Ingress Resource**:

```bash
kubectl apply -f wear-watch-path-ingress.yaml
```

**Describe the Ingress Resource**:

```bash
kubectl describe ingress wear-watch-path-ingress
```

```
Name:             wear-watch-path-ingress
Namespace:        default
Address:
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host   Path   Backends
  ----   ----   --------
  *
         /wear    wear-service:80   (<none>)
         /watch   watch-service:80  (<none>)
Annotations:  <none>
Events:       <none>
```

The `Default backend` entry handles traffic that does not match any defined path. A separate Service must be deployed to serve responses for unmatched paths, such as a custom 404 page for `/listen`.

---

### Host-Based Routing

To route traffic based on the DNS hostname, specify a `host` field in each rule:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wear-watch-host-ingress
spec:
  rules:
    - host: "wear.my-online-store.com"
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80
    - host: "watch.my-online-store.com"
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 80
```

Multiple DNS entries can point to the same Ingress Controller Service IP, allowing different hostnames to reach the same cluster. Within a hostname rule, multiple paths can be defined to further subdivide traffic across backend Services.

---

## Path-Based vs. Host-Based Routing

|                | Path-Based                       | Host-Based       |
|----------------|----------------------------------|------------------|
| Rules          | 1                                | Multiple         |
| Paths per rule | Multiple                         | 1 or more        |
| Use when       | Single domain, multiple services | Multiple domains |

Rules operate at two levels: the **host level** matches the domain name, and the **path level** matches the URL path and routes to the appropriate backend Service. Traffic that matches no rule or path is sent to the default backend.

---

## Conclusion

Instead of managing external routing, and path rules, separately across multiple Services or external configurations, Ingress consolidates them into a single declarative resource.

| Concept                   | Details                                                                         |
|---------------------------|---------------------------------------------------------------------------------|
| **Ingress Controller**    | Must be deployed manually; not included by default                              |
| **Ingress Resource**      | A manifest containing routing rules read by the controller                      |
| **Supported controllers** | GCE, nginx (maintained by Kubernetes project), Contour, HAProxy, Traefik, Istio |
| **Routing strategies**    | Default backend, path-based, host-based                                         |
| **Default backend**       | Handles unmatched traffic; must be deployed as a real Service                   |
| **External exposure**     | Requires a NodePort or LoadBalancer Service for the controller itself           |

---
