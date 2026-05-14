---
title: Introduction Network Policies
date: 2026-03-17
weight: 5
tags: [Network Policies]
---
# Network Policies

Before diving into Kubernetes network concepts, it helps to understand the basics of network traffic directions. Consider a simple three-tier web application:

- A **WEB** server on port `80` serves as the front end to users.
- An **API** server on port `5000` serving back-end APIs.
- A **DB** database server on port `3306`.

The user sends a request to the WEB server on port `80`. The WEB server sends a request to the API server on port `5000`. The API server fetches data from the DB server on port `3306` and returns the result to the user via the WEB server.

Traffic flows in two directions:

- **Ingress** for incoming traffic to a component.
- **Egress** for outgoing traffic from the component.

The response to a request does not factor into this classification; only the direction the traffic originated matters.

```
      User  
    |      ▲ 
    |      │
  ┌─▼──────│─┐  Egress   ┌──────────┐  Egress   ┌────────┐
  │          │──────────▶│          │──────────▶│        │
  │   WEB    │           │   API    │           │   DB   │
  │          │◀- - - - - │          │◀- - - - - │        │
  └──────────┘  Ingress  └──────────┘  Ingress  └────────┘
```

### Required Rules

To get this three-tier application working, the following rules are needed:

- **Ingress** on the WEB server: accept HTTP traffic on port `80`
- **Egress** from the WEB server: allow traffic to port `5000` on the API server
- **Ingress** on the API server: accept traffic on port `5000`
- **Egress** from the API server: allow traffic to port `3306` on the DB server
- **Ingress** on the DB server: accept traffic on port `3306`

---

## Network Security in Kubernetes

Each Node, Pod, and Service in a Kubernetes cluster has its own IP address. A core requirement of Kubernetes networking is that all Pods can communicate with each other without additional configuration, such as routes. By default, Kubernetes applies an **all-allow** rule that permits traffic between any Pod or Service within the cluster.

Mapping the three-tier example onto Kubernetes, each component gets its own Pod: one for the WEB server, one for the API server, and one for the DB server. By default, all three Pods communicate freely. A security requirement demands that the WEB Pod cannot communicate directly with the DB Pod. This is where a Network Policy comes in.

---

## Network Policies

A NetworkPolicy is a Kubernetes resource that defines rules for which traffic is allowed to and from particular Pods. For example, you might apply a Network Policy to the DB Pod that only allows ingress traffic from the API Pod on port `3306`. After this policy is created, any other traffic to that Pod is blocked, and only traffic matching the rule is allowed through.

### Linking a Network Policy to a Pod

Network Policies are linked to Pods using **labels and selectors**, the same technique used by Deployments and Services. The Pod is labeled, and this label is referenced in the `podSelector` field of the Network Policy. The `policyTypes` field specifies whether the rule controls Ingress, Egress, or both.

### Ingress Policy

The following manifest applies a Network Policy to the DB Pod, allowing only Ingress traffic from the API Pod on port `3306`:

**Create a Network Policy for the DB Pod**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-np
spec:
  podSelector:
    matchLabels:
      pod: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              pod: api
      ports:
        - protocol: TCP
          port: 3306
```

**Apply the policy**
```bash
kubectl create -f db-policy.yaml
```

### Ingress and Egress Policy

To restrict outbound traffic from the DB Pod, add `Egress` to `policyTypes` and define an egress rule:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-egress-np
spec:
  podSelector:
    matchLabels:
      pod: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              pod: api
      ports:
        - protocol: TCP
          port: 3306
  egress:
    - to:
        - podSelector:
            matchLabels:
              pod: api
      ports:
        - protocol: TCP
          port: 5000
```

This ensures the DB Pod receives traffic only from the API Pod on port `3306` and sends only traffic to the API Pod on port `5000`. All other inbound and outbound traffic is blocked.

---

## Network Solution Support

Network Policies are enforced by the CNI installed in the cluster; not all of them support this. CNIs that support Network Policies include Cilium, Calico, Kube-router, Romana, and Weave Net. Flannel does not support Network Policies.

Even in a cluster using a CNI that does not support Network Policies, the policy objects can still be created without error. They will not be enforced, and no error is produced to indicate the lack of support. Always verify compatibility with the chosen CNI before relying on Network Policies for security.

---
