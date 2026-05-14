---
title: Gateway API
date: 2026-03-17
weight: 4
tags: [Gateway API]
---
# Introduction to Gateway API

The Kubernetes Ingress resource plays a fundamental role in managing external requests to Services running in a cluster. Ingress routes traffic to Services based on the path and host specified in the request. It has been widely adopted, but its capabilities are limited to routing requirements.

To implement more advanced traffic-routing logic, teams rely on vendors that provide Ingress Controllers. These vendors use annotations to extend Ingress functionality, which makes configurations incompatible (vendor-locking) across different controllers and implementations:

```yaml
# nginx
nginx.ingress.kubernetes.io/ssl-redirect: "true"
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

# Traefik
traefik.ingress.kubernetes.io/headers.customresponseheaders: |
  Access-Control-Allow-Origin: '*'
```

Kubernetes isn't aware of these annotations and can't validate them, making Ingress harder to maintain and collaborate on.

The Kubernetes Gateway API was introduced to address the problems caused by vendor-specific annotations and the lack of a common standard for Ingress. Gateway API provides advanced traffic-management capabilities natively, without depending on custom annotations.

---

## What Is Gateway API?

Gateway API is an official Kubernetes project focused on Layer 4 and Layer 7 routing. It represents the next generation of Kubernetes Ingress, load balancing, and Service Mesh APIs, designed to be expressive, extensible, and role-oriented.

Gateway API introduces three distinct objects, each managed by a different persona:

| Persona                 | Object                                     | Responsibility                                                                                                                               |
|-------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Infrastructure Provider | `GatewayClass`                             | Defines the underlying network infrastructure (nginx, Traefik, etc.). Analogous to a `StorageClass` in the storage world.                    |
| Cluster Operator        | `Gateway`                                  | An instance of a `GatewayClass`. Configures listeners, ports, and TLS settings. Operators control the infrastructure; developers consume it. |
| Application Developer   | `HTTPRoute`, `TCPRoute`, `GRPCRoute`, etc. | Defines routing rules for their application. Unlike Ingress, Gateway API supports a full range of protocols at this layer.                   |

This separation means each team works within their own object. There is no shared resource to coordinate over, and no risk of one team's change breaking another team's configuration.

---

## The Core Objects

### GatewayClass

Comparable to a `StorageClass`, It defines a group of Gateways that share the same configuration and are managed by a controller that complies with the class's specifications. The `controllerName` field links the `GatewayClass` to the deployed controller.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium-gateway-controller
spec:
  controllerName: io.cilium/gateway-controller
```

### Gateway

References the `GatewayClass` and declares its listeners. A controller must be deployed to implement Gateway API, just as with Ingress.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

A real-world example using Cilium with routes allowed from all Namespaces:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  namespace: kube-system
  name: cilium-gateway
  annotations:
    io.cilium/lb-ipam-ips: "192.168.3.201"
spec:
  gatewayClassName: cilium-gateway-controller
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

### HTTPRoute

Attaches to a Gateway resource and defines routing rules so that incoming HTTP traffic with a specific hostname and path prefix is routed from the Gateway to the designated Service and port.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mealie-http
  namespace: mealie
spec:
  parentRefs:
    - name: cilium-gateway
      namespace: kube-system
  hostnames:
    - "mealie.homelab.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: mealie-svc
          port: 9000
```

---

## Solving Real Ingress Limitations

### TLS and HTTPS Redirects

With Ingress, redirecting HTTP traffic to HTTPS requires a controller-specific annotation:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - secure.example.com
      secretName: tls-secret
```

Gateway API handles this declaratively in the `Gateway` spec, with no annotations required:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: secure-gateway
spec:
  gatewayClassName: example-gc
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: tls-secret
      allowedRoutes:
        kinds:
          - kind: HTTPRoute
```

### Traffic Splitting

With nginx Ingress, canary deployments rely on annotations that route a percentage of traffic to another Service. The relationship between the canary and the primary Ingress is implicit:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-v2
                port:
                  number: 80
```

Gateway API expresses the same intent in a single, self-contained configuration. Both backend Services are listed explicitly with their weights:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: split-traffic
spec:
  parentRefs:
    - name: app-gateway
  rules:
    - backendRefs:
        - name: app-v1
          port: 80
          weight: 80
        - name: app-v2
          port: 80
          weight: 20
```

A complete example routing all traffic to a frontend service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
      namespace: nginx-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend-svc
          port: 80
```

### CORS and Header Manipulation

With Ingress, CORS settings require controller-specific annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-ingress
  annotations:
    traefik.ingress.kubernetes.io/headers.customresponseheaders: |
      Access-Control-Allow-Origin: '*'
      Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
      Access-Control-Allow-Headers: Content-Type, Authorization
      Access-Control-Allow-Credentials: true
      Access-Control-Max-Age: 3600
```

Gateway API configures this directly in the route spec using the `ResponseHeaderModifier` filter:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cors-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: Access-Control-Allow-Origin
                value: "*"
              - name: Access-Control-Allow-Methods
                value: "GET, POST, PUT, DELETE, OPTIONS"
              - name: Access-Control-Allow-Headers
                value: "Content-Type, Authorization"
              - name: Access-Control-Allow-Credentials
                value: "true"
              - name: Access-Control-Max-Age
                value: "3600"
      backendRefs:
        - name: api-service
          port: 80
```

---

## Supported Route Types

`HTTPRoute` used Layer 7, but the Gateway API supports a broader set of protocols. Each route type targets a specific layer or protocol:

| Route Type  | Layer   | Use Case                                                                          |
|-------------|---------|-----------------------------------------------------------------------------------|
| `HTTPRoute` | Layer 7 | HTTP and HTTPS traffic routing based on host, path, headers, and query parameters |
| `GRPCRoute` | Layer 7 | gRPC-specific routing based on service and method names                           |
| `TLSRoute`  | Layer 4 | TLS traffic routing based on SNI hostname, without terminating TLS                |
| `TCPRoute`  | Layer 4 | Raw TCP traffic routing when no higher-level protocol is available                |
| `UDPRoute`  | Layer 4 | UDP traffic routing for protocols such as DNS or gaming                           |

The key difference between route types is the level at which they operate. `HTTPRoute` and `GRPCRoute` can inspect request content such as headers and paths. `TLSRoute`, `TCPRoute`, and `UDPRoute` operate at the transport layer and route based on connection properties only, not request content.

This range is one of the most significant advantages over Ingress, which has no native support for non-HTTP protocols.

---

## Implementation Status

Gateway API defines three conformance levels, which describe how broadly a controller implements the specification:

| Level                       | Description                                                                  |
|-----------------------------|------------------------------------------------------------------------------|
| **Core**                    | Mandatory features that every conformant implementation must support         |
| **Extended**                | Optional features that implementations may support for broader functionality |
| **Implementation-specific** | Custom features that go beyond the Gateway API specification                 |

When choosing a controller, check its conformance report to understand which features are supported. The Gateway API project publishes conformance results at [gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io/implementations/).

Most major controllers have implemented the Gateway API or are actively working toward it:

- Amazon EKS (AWS Load Balancer Controller)
- Azure Application Gateway for Containers
- Contour
- Envoy Gateway
- Google Kubernetes Engine
- HAProxy Kubernetes Ingress Controller
- Istio
- nginx Gateway Fabric
- Traefik
- Cilium

To verify whether your controller supports Gateway API, check its documentation or run:

```bash
kubectl get crd gatewayclasses.gateway.networking.k8s.io
```
```
NAME                                       CREATED AT
gatewayclasses.gateway.networking.k8s.io   2026-03-25T17:06:21Z
```

If the CRD exists, Gateway API is installed in the cluster. To check which GatewayClasses are available:

```bash
kubectl get gatewayclass
```
```
NAME                        CONTROLLER                     ACCEPTED   AGE
cilium-gateway-controller   io.cilium/gateway-controller   True       49d
```

The ecosystem has converged on Gateway API as the standard going forward.

---
