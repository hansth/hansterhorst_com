---
title: Service
weight: 1
tags: [Services]
---
# Services

A Kubernetes Service provides a stable endpoint for exposing applications running across Pods, either within the cluster or externally.

Each Pod gets its own internal IP address, but Pods are ephemeral. When a Pod restarts or is rescheduled, it receives a new IP address. A Service solves this by providing:

- **Stable IP address**: Remains constant even when Pods are recreated.
- **Load balancing**: Distributes requests across multiple Pods.
- **Loose coupling**: Abstracts communication both inside and outside the cluster.

---

## Service Types

Kubernetes offers four primary Service types:

| Type           | Scope               | External Access       | Use Case                            |
|----------------|---------------------|-----------------------|-------------------------------------|
| `ClusterIP`    | Internal            | No                    | Inter-Pod communication             |
| `NodePort`     | Internal + External | Via `NodeIP:NodePort` | Development, simple external access |
| `LoadBalancer` | Internal + External | Via LB external IP    | Production external services        |
| `ExternalName` | N/A                 | N/A                   | DNS aliasing for external services  |

The **Headless Service** is a special type of ClusterIP that returns individual Pod IPs via DNS instead of a single virtual IP. It is created by setting `spec.clusterIP: None`.

---

## ClusterIP

The default Service type. Creates a Service with an internal IP address accessible only within the cluster. Ideal for communication between application components.

**Create a Deployment**
```bash
kubectl create deployment nginx \
  --image=spurin/nginx-debug \
  --port=80 --replicas=3 \
  -o yaml --dry-run=client > deployment.yaml
```

**Apply the Deployment**
```bash
kubectl apply -f deployment.yaml
```

**Generate a Service manifest**
```bash
kubectl expose deployment/nginx \
  --name=nginx-svc -o yaml | tee nginx-svc.yaml
```

The Service uses a selector matching the `app: nginx` label to identify the correct Pods and retrieve the port configuration from the Deployment.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx-svc
  namespace: default
spec:
  clusterIP: 10.98.126.149
  clusterIPs:
  - 10.98.126.149
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: ClusterIP
```

**Verify the Service**
```bash
kubectl get service
```
```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-svc  ClusterIP   10.98.126.149   <none>        80/TCP    9s
```

### Endpoints

Kubernetes automatically creates an Endpoints object alongside the Service. It maintains the IP addresses of the Pods the Service routes traffic to.

**List endpoints**
```bash
kubectl get endpoints
```
```
NAME         ENDPOINTS                                   AGE
nginx-svc    10.0.0.177:80,10.0.1.153:80,10.0.3.229:80   2m54s
```

**Verify Pod IPs**
```bash
kubectl get pods -o wide
```
```
NAME                          READY   STATUS    IP           NODE
nginx-857c8784b8-2crbt        1/1     Running   10.0.3.229   k8s-master-1
nginx-857c8784b8-gb9hw        1/1     Running   10.0.0.177   k8s-worker-1
nginx-857c8784b8-tz44c        1/1     Running   10.0.1.153   k8s-master-2
```

**Describe the Service**
```bash
kubectl describe svc nginx-svc
```
```
Name:                     nginx-svc
Namespace:                default
Labels:                   app=nginx
Annotations:              <none>
Selector:                 app=nginx
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.98.126.149
IPs:                      10.98.126.149
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                10.0.0.177:80,10.0.3.229:80,10.0.1.153:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
```
- `Selector: app=nginx`: Selects all Pods with this label.
- `Type: ClusterIP`: Internal-only Service, not exposed outside the cluster.
- `IP: 10.98.126.149`: Stable virtual IP address.
- `Port: 80/TCP`: The port the Service listens on.
- `TargetPort: 80/TCP`: The port on the Pod where traffic is forwarded. Traffic flow: `Service:80` → `Pod:80`.
- `Endpoints`: The actual Pod IPs backing this Service. Traffic is load-balanced across all three.

**Filter the Pods behind the Service**
```bash
kubectl get pods --selector=app=nginx
```
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-857c8784b8-2crbt   1/1     Running   0          10m
nginx-857c8784b8-gb9hw   1/1     Running   0          10m
nginx-857c8784b8-tz44c   1/1     Running   0          10m
```

Rather than connecting directly to the Pods' ephemeral IP addresses, clients target the stable ClusterIP (`10.98.126.149`) or the DNS name (`nginx-svc.default.svc.cluster.local`). Kubernetes then routes traffic across the available, healthy Pods behind the Service.

### Accessing a ClusterIP Service

**Internal access** — from a cluster Node, curl the ClusterIP directly:

```bash
curl 10.98.126.149:80
```
```
<!DOCTYPE html>
<body>
<p><em>Hostname: nginx-857c8784b8-tz44c</em></p>
<p><em>IP Address: 10.0.1.153:80</em></p>
<p><em>URL: /</em></p>
<p><em>Request Method: GET</em></p>
<p><em>Request ID: edd270722affa3b7508d58b4078d0ece</em></p>
</body>
</html>
```

Repeating the command demonstrates load balancing across different Pod instances:

```
Hostname: nginx-857c8784b8-2crbt
IP Address: 10.0.3.229:80

Hostname: nginx-857c8784b8-gb9hw
IP Address: 10.0.0.177:80
```

**External access (development only)** — use port forwarding to access the Service from outside the cluster:

```bash
kubectl port-forward service/nginx-svc 8080:80
```

This forwards local port `8080` to the Service on port `80`, making it accessible at `http://localhost:8080`. Port forwarding works regardless of Service type and is useful for local development and testing.

### DNS-Based Service Discovery

One of the most powerful features of Kubernetes Services is its integration with the cluster's CoreDNS, enabling name-based discovery rather than IP addresses.

**Run a temporary curl Pod**
```bash
kubectl run test --image=curlimages/curl \
  --restart=Never --rm -it \
  -- sh
```

**Access the Service using its fully qualified domain name**
```bash
curl nginx-svc.default.svc.cluster.local
```

This follows Kubernetes' standard DNS naming convention: `<service-name>.<namespace>.svc.cluster.local`.

**Inspect the DNS search paths**
```bash
cat /etc/resolv.conf
```
```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

Because `default.svc.cluster.local` is included in the search path, the Service is also accessible using just its name:

```bash
curl nginx-svc
```

Kubernetes automatically expands this to the full DNS name.

> **Note**: ClusterIP is only accessible inside the cluster.

---

## NodePort

While a ClusterIP Service is only accessible from within the cluster, a NodePort Service extends ClusterIP by adding external access. When Node IP addresses are externally reachable, NodePort Services become accessible from outside the cluster.

**Modify the Service manifest**
```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
```

The `kube-proxy` allocates a port from the `30000-32767` range, exposed on every Node.

**Reapply the manifest**
```bash
kubectl apply -f nginx-svc.yaml
```

**Verify the Service**
```bash
kubectl get service
```
```
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-svc   NodePort   10.98.126.149   <none>        80:31994/TCP   52m
```

The `PORT(S)` column shows `80:31994/TCP`:

- `80`: The ClusterIP port forwarding to `targetPort` on the Pod.
- `31994`: The NodePort, exposed on every cluster Node in the `30000-32767` range.

**List Nodes with their IP addresses**
```bash
kubectl get nodes -o wide
```
```
NAME           STATUS   ROLES           VERSION   INTERNAL-IP
k8s-master-1   Ready    control-plane   v1.31.0   192.168.3.110
k8s-master-2   Ready    control-plane   v1.31.0   192.168.3.120
k8s-master-3   Ready    control-plane   v1.31.0   192.168.3.130
k8s-worker-1   Ready    <none>          v1.31.0   192.168.3.140
```

**Internal access**

```
Pod → ClusterIP:80 → targetPort:80 → Pod containerPort:80
```
```bash
curl 10.98.126.149:80
```

**External access**

```
Client → NodeIP:31994 → Service port:80 → targetPort:80 → Pod containerPort:80
```
```bash
curl 192.168.3.140:31994
```

The request works on any Node IP, even if the Pod is not running on that Node. `kube-proxy` handles routing automatically.

---

## LoadBalancer

LoadBalancer builds on NodePort by adding a cloud-native external load balancer. Availability depends on the environment: cloud providers typically include built-in support, while on-premises deployments may require solutions such as MetalLB or Cilium.

**Modify the Service manifest**
```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
```

Or delete the existing Service and create a new one imperatively:

```bash
kubectl delete service/nginx-svc

kubectl expose deployment/nginx --name=nginx-svc --type=LoadBalancer
```
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx-svc
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

**Verify the Service**
```bash
kubectl get service
```
```
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx-svc   LoadBalancer   10.104.237.29   192.168.3.202   80:30426/TCP   25s
```
- `80`: LoadBalancer and ClusterIP port.
- `30426`: NodePort, still accessible as a fallback.
- `EXTERNAL-IP`: The load balancer's external IP address.

**Test the LoadBalancer endpoint**
```bash
watch "curl 192.168.3.202:80 2>/dev/null"
```

This executes every 2 seconds, curling the load balancer IP and distributing requests across available Pods.

**Scale the Deployment**
```bash
kubectl scale deployment/nginx --replicas=10
```

As the Deployment scales, the Endpoints object updates automatically and the Service routes traffic to all available Pods without manual intervention.

### Traffic Flow

LoadBalancer builds on both NodePort and ClusterIP:
```
LoadBalancer (external IP) → NodePort → ClusterIP → Pod
```

All three access methods work simultaneously:

```bash
# Via LoadBalancer external IP
curl 192.168.3.202:80

# Via NodePort on any Node
curl 192.168.3.140:30426

# Via ClusterIP (internal only)
curl 10.104.237.29:80
```

---

## ExternalName

ExternalName maps a Service name to an external DNS name. Unlike other Service types, it doesn't forward traffic or create endpoints. Instead, it returns a `CNAME` record that redirects to an external service.

**Create two Deployments with different styling**
```bash
kubectl create deployment nginx-red \
  --image=spurin/nginx-red \
  --port=80 \
  --replicas=3

kubectl create deployment nginx-blue \
  --image=spurin/nginx-blue \
  --port=80 \
  --replicas=3
```

**Expose both as ClusterIP Services**
```bash
kubectl expose deployment/nginx-red
kubectl expose deployment/nginx-blue
```
```
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-blue  ClusterIP  10.96.67.110    <none>        80/TCP    5s
nginx-red   ClusterIP  10.106.16.218   <none>        80/TCP    9s
```

Both Services need to be exposed so the ExternalName Service can reference them by their DNS names.

**Create an ExternalName Service aliasing `nginx-red`**
```bash
kubectl create service externalname externalname-svc \
  --external-name=nginx-red.default.svc.cluster.local \
  -o yaml | tee externalname-svc.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: externalname-svc
  name: externalname-svc
  namespace: default
spec:
  externalName: nginx-red.default.svc.cluster.local
  sessionAffinity: None
  type: ExternalName
```

**Apply the manifest**
```bash
kubectl apply -f externalname-svc.yaml
```

While this uses an internal cluster DNS name, ExternalName Services can also point to external domains such as `database.example.com`, making them useful for integrating external resources into cluster service discovery.

### Testing the ExternalName Service

**Launch a temporary curl container**
```bash
kubectl run curl -it --rm \
  --image=curlimages/curl \
  --restart=Never -- sh
```

**Access each Service directly**
```bash
curl nginx-red
curl nginx-blue
```

**Access via the alias**
```bash
curl externalname-svc
```

The response comes from `nginx-red` because `externalname-svc` is a CNAME alias pointing to it.

**Verify the DNS mapping**
```bash
nslookup externalname-svc
```
```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

externalname-svc.default.svc.cluster.local
canonical name = nginx-red.default.svc.cluster.local
Name:   nginx-red.default.svc.cluster.local
Address: 10.106.16.218
```

`externalname-svc` is a CNAME record pointing to `nginx-red.default.svc.cluster.local`, which resolves to its ClusterIP.

### Key Characteristics

- No ClusterIP assigned.
- No Endpoints object created.
- No port mapping or load balancing.
- Pure DNS-level redirection via CNAME record.
- No `kube-proxy` involvement.

**Use cases**: external databases (AWS RDS, Azure SQL), third-party APIs, migrating services between internal and external without changing application code.

---

## Headless Services

A Headless Service is a special type of ClusterIP created by setting `clusterIP: None`. Instead of providing a single virtual IP, it works like round-robin DNS, returning multiple Pod IP addresses in response to DNS queries.

**Create a Deployment**
```bash
kubectl create deployment nginx \
  --image=spurin/nginx-debug \
  --port=80 \
  --replicas=2
```

**Generate a Service manifest**
```bash
kubectl expose deployment nginx \
  --dry-run=client -o yaml \
  > headless-svc.yaml
```

**Add `clusterIP: None` to the spec**
```yaml
spec:
  clusterIP: None
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
```

**Apply the manifest**
```bash
kubectl apply -f headless-svc.yaml
```

**Verify the Service**
```bash
kubectl get service
```

```
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
nginx   ClusterIP   None         <none>        80/TCP
```

The `CLUSTER-IP` column shows `None`, confirming no virtual IP is assigned.

### Testing Headless Service Behavior

**Launch a curl container**
```bash
kubectl run curl -it --rm \
  --image=curlimages/curl \
  --restart=Never -- sh
```

**Test DNS resolution**
```bash
nslookup nginx
```

Standard ClusterIP returns a single virtual IP:

```
Server:   10.96.0.10
Address:  10.96.0.10:53
Name:     nginx.default.svc.cluster.local
Address:  10.111.42.24
```

Headless Service returns all Pod IPs:

```
Server:   10.96.0.10
Address:  10.96.0.10:53
Name:     nginx.default.svc.cluster.local
Address:  10.0.1.131
Name:     nginx.default.svc.cluster.local
Address:  10.0.0.69
```

**Test with curl**
```bash
curl nginx
```

Repeated requests are routed to different Pods based on DNS round-robin:

```
<p><em>Hostname: nginx-857c8784b8-wd8gx</em></p>
<p><em>IP Address: 10.0.1.131:80</em></p>

<p><em>Hostname: nginx-857c8784b8-mgwfg</em></p>
<p><em>IP Address: 10.0.0.69:80</em></p>
```

**Clean up**
```bash
exit
kubectl delete -f headless-svc.yaml
kubectl delete deployment/nginx
```

### Key Difference from Other Service Types

|                 | ClusterIP / NodePort / LoadBalancer | ExternalName                  | Headless                      |
|-----------------|-------------------------------------|-------------------------------|-------------------------------|
| Traffic routing | Via `kube-proxy`                    | DNS CNAME to external service | DNS returns Pod IPs directly  |
| Virtual IP      | Yes                                 | No                            | No                            |
| Use case        | Standard workloads                  | External service aliasing     | Stateful apps, peer discovery |

Headless Services are ideal for stateful applications such as databases, peer discovery scenarios, and cases that require direct Pod-to-Pod communication without proxy overhead.

---

## Traffic Flow Summary

```
ClusterIP:    Pod → ClusterIP:port → targetPort → containerPort
NodePort:     Client → NodeIP:NodePort → Service:port → targetPort → containerPort
LoadBalancer: Client → LB IP:port → NodePort → Service:port → targetPort → containerPort
ExternalName: Pod → DNS CNAME → External Service
Headless:     Pod → DNS (returns Pod IPs) → Direct Pod IP:containerPort
```

---

## Port Terminology

| Term            | Description                                          |
|-----------------|------------------------------------------------------|
| `containerPort` | Port the application listens on inside the container |
| `targetPort`    | Port on the Pod where the Service forwards traffic   |
| `port`          | Port the Service listens on                          |
| `nodePort`      | External port exposed on all Nodes (30000-32767)     |

---

## DNS Naming Convention

```
<service-name>.<namespace>.svc.cluster.local
```
- `nginx.default.svc.cluster.local` — full FQDN
- `nginx.default` — cross-namespace shorthand
- `nginx` — same namespace shorthand

---

## Conclusion

The four primary Kubernetes Service types:

1. **ClusterIP**: Internal cluster communication with a stable virtual IP.
2. **NodePort**: External access through ports on cluster Nodes.
3. **LoadBalancer**: Cloud-native load balancing with an external IP address.
4. **ExternalName**: DNS aliasing for internal and external resources.

Kubernetes Services provide stable endpoints for dynamically changing Pods, enable service discovery via DNS, and support flexible patterns for both internal and external access.