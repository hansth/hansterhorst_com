---
title: Practical Gateway API
date: 2026-03-17
weight: 4
tags: [Gateway API]
---
# A Practical Guide Using Gateway API

This guide demonstrates the Kubernetes Gateway API using the NGINX Gateway Controller. The concepts and APIs are implementation-agnostic and apply across any Gateway API-compatible controller.

**Official docs**: [gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io/)

---

## Installing Gateway API with NGINX

The Gateway API defines CRDs, but a controller is needed to implement them. This guide uses the NGINX Gateway Controller, which supports all standard Gateway API resources.

**Install the Gateway API CRDs**
```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.2" | kubectl apply -f -

kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/experimental?ref=v1.6.2" | kubectl apply -f -
```

**Install the NGINX Gateway Controller**
```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n nginx-gateway
```

**Reference**: [NGINX Gateway Fabric](https://docs.nginx.com/nginx-gateway-fabric/)

---

## GatewayClass

A `GatewayClass` defines a set of Gateways managed by a specific controller. It decouples Gateway configuration from the underlying implementation, and multiple GatewayClasses can coexist in the same cluster, for example, nginx and Istio side by side.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/gateway-controller
```

The `controllerName` field must match the name expected by the controller. For NGINX, that is `nginx.org/gateway-controller`.

**Reference**: [GatewayClass](https://gateway-api.sigs.k8s.io/reference/api-types/gatewayclass/)

---

## Gateway

A `Gateway` defines how traffic enters the cluster. It specifies the protocols, ports, and which routes are allowed to attach.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

- `gatewayClassName`: References the `GatewayClass` that manages this Gateway.
- `listeners.protocol`: The protocol this listener handles.
- `listeners.port`: The port on which the Gateway listens.
- `allowedRoutes.namespaces.from: All`: Permits routes from any Namespace to attach to this Gateway.

**Reference**: [Gateway](https://gateway-api.sigs.k8s.io/reference/api-types/gateway/)

---

## HTTPRoute

An `HTTPRoute` defines how HTTP traffic is forwarded to Services. It attaches to a Gateway and routes requests based on rules such as path prefixes or headers.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: basic-route
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /app
      backendRefs:
        - name: my-app
          port: 80
```
- `parentRefs`: Attaches this route to the named Gateway.
- `rules.matches.path`: Matches requests whose path begins with `/app`.
- `backendRefs`: Forwards matched traffic to `my-app` on port `80`.

**Reference**: [HTTPRoute](https://gateway-api.sigs.k8s.io/reference/api-types/httproute/)

---

## HTTP Redirects and Rewrites

### HTTP to HTTPS Redirect

The `RequestRedirect` filter redirects all HTTP requests to HTTPS:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
```

### Path Rewrite

The `URLRewrite` filter modifies the request path before it reaches the backend. Requests matching `/old` have that prefix replaced with `/new`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-path
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /old
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              replacePrefixMatch: /new
      backendRefs:
        - name: my-app
          port: 80
```

**Reference**: [HTTP Redirects and Rewrites](https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/)

---

## HTTP Header Modification

The `RequestHeaderModifier` filter adds, sets, or removes HTTP headers on requests before they reach the backend. This is useful for tagging requests with environment-specific metadata:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-mod
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: x-env
                value: staging
      backendRefs:
        - name: my-app
          port: 80
```

**Reference**: [HTTP Header Modification](https://gateway-api.sigs.k8s.io/guides/http-header-modifier/)

---

## Traffic Splitting

The `weight` field on each `backendRef` controls the proportion of traffic each Service receives. This is commonly used for canary deployments and A/B testing:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: traffic-split
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - backendRefs:
        - name: v1-service
          port: 80
          weight: 80
        - name: v2-service
          port: 80
          weight: 20
```

80% of requests go to `v1-service` and 20% go to `v2-service`.

**Reference**: [Traffic Splitting](https://gateway-api.sigs.k8s.io/guides/traffic-splitting/)

---

## Request Mirroring

The `RequestMirror` filter sends a copy of each incoming request to a secondary Service without affecting the primary response. The original request is forwarded to `my-app` as usual, while a copy is silently sent to `mirror-service`. Useful for shadow testing and traffic analysis:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: request-mirror
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - filters:
        - type: RequestMirror
          requestMirror:
            backendRef:
              name: mirror-service
              port: 80
      backendRefs:
        - name: my-app
          port: 80
```

**Reference**: [Request Mirroring](https://gateway-api.sigs.k8s.io/guides/http-request-mirroring/)

---

## TLS Termination

Kubernetes Gateways can terminate TLS at the Gateway level using a certificate stored in a Kubernetes `Secret`, decrypting traffic before forwarding it to backend Services as plain HTTP:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway-tls
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: tls-secret
      allowedRoutes:
        namespaces:
          from: All
```
- `protocol: HTTPS`: This listener handles encrypted HTTPS traffic.
- `tls.mode: Terminate`: The Gateway decrypts incoming TLS traffic before forwarding it to backends.
- `certificateRefs`: Points to a `Secret` containing the TLS certificate and private key.

**Reference**: [TLS Configuration](https://gateway-api.sigs.k8s.io/guides/tls/)

---

## TCP, UDP, and gRPC

Gateway API supports protocols beyond HTTP, making it suitable for database, DNS, and microservice communication.

### TCP

TCP is a connection-oriented protocol commonly used for database access. The following example exposes a MySQL service on port `3306`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tcp-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: tcp
      protocol: TCP
      port: 3306
      allowedRoutes:
        namespaces:
          from: All
```

**Reference**: [TCP Routing](https://gateway-api.sigs.k8s.io/guides/user-guides/tcp/)

### UDP

UDP is a connectionless protocol used for DNS and streaming. The following example exposes a DNS service on port `53`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: udp-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: udp
      protocol: UDP
      port: 53
      allowedRoutes:
        namespaces:
          from: All
```

### gRPC

gRPC is a high-performance RPC framework used in microservices. Use `GRPCRoute` to route gRPC traffic based on service and method names:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-route
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
    - matches:
        - method:
            service: my.grpc.Service
            method: GetData
      backendRefs:
        - name: grpc-service
          port: 50051
```

The `method.service` and `method.method` fields match a specific gRPC service and method. Matched requests are forwarded to `grpc-service` on port `50051`.