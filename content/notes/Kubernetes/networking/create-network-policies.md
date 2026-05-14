---
title: Create Network Policies
date: 2026-03-17
weight: 6
tags: [Network Policies]
---
# Create Network Policies

Network policies are Kubernetes resources used to control traffic to and from Pods at the IP address and port level (OSI layers 3 and 4). They define which communications are allowed between Pods and between Pods and external endpoints.

---

## The default behavior

By default, Kubernetes is completely open. Every Pod can communicate with every Pod across all Namespaces without any restrictions. This is called the flat network model. Network Policies let you control this by selectively restricting traffic between Pods.

---

## Key concept: network policies are additive

A Network Policy selects Pods and defines what traffic they are allowed to receive or send. Once a Pod has a Network Policy, only traffic that matches that policy is allowed, and all other traffic is denied. When multiple policies are applied to the same Pod, those rules are combined:

- **No policies**: all traffic allowed
- **One policy selects a Pod**: only matching traffic allowed, everything else denied
- **Multiple policies selecting a Pod**: union of all allowed traffic (additive)

---

## Kubernetes Network Policies in depth

To see how this works in practice, consider three Pods:

- WEB Pod, web server on port `80`
- API Pod, API server on port `5000`
- DB Pod, database server on port `3306`

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

The goal is to protect the DB Pod so that only the API Pod can access it on port `3306`. The WEB and API Pods do not require any restrictions.

### Blocking all Traffic First

Kubernetes allows all traffic by default, so the first step is to override that for the DB Pod. Create a `NetworkPolicy` manifest called `db-network-policy` and bind it to the DB Pod using `podSelector` with the label `pod: db`. This is enough to block all traffic to and from the DB Pod.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
spec:
  podSelector:
    matchLabels:
      pod: db
```

With the DB Pod isolated, the next step is to decide what traffic to let back in.

### Choosing the Right Policy Type: Ingress or Egress

There are two rule types: `Ingress` for incoming traffic and `Egress` for outgoing traffic.

> **Note**: Always think from the perspective of the Pod being protected.

The API Pod sends queries to the DB Pod. From the DB Pod's point of view, that is incoming traffic, so the rule type is `Ingress`. The response back to the API Pod does not need a separate rule, and once an ingress connection is permitted, the response is automatically allowed through.

A single `Ingress` rule is all that is needed here.

### Writing the Ingress Rule

An `ingress` rule has two fields: `from` defines who is allowed in, and `ports` defines which port they can reach. Add `policyTypes: [Ingress]` to the manifest and an `ingress` section below it.

The `from` field uses a `podSelector` with `matchLabels` to identify the API Pod. The `ports` field specifies port `3306` with protocol `TCP`. Now the DB Pod only accepts traffic from the API Pod on port `3306`, and everything else remains blocked.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
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

This works fine when all Pods are in the same namespace. But what happens when they are not?

### Restricting by Namespace

In a cluster with separate Namespaces for `dev`, `test`, and `prod`, all three API Pods share the same labels. The current rule would allow all of them to reach the DB Pod. The goal is to allow only the `prod` API Pod.

The fix is to add a `namespaceSelector` inside the same list entry as the `podSelector`. Both selectors sharing a single entry mean AND, the Pod must match both conditions simultaneously. The Namespace itself must be labeled first for this to work.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
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
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: prod
      ports:
        - protocol: TCP
          port: 3306
```

If only a `namespaceSelector` is provided with no `podSelector`, all Pods in the matching Namespace are allowed Ingress. Now the DB Pod is locked down to the right Pod in the right Namespace. But what about traffic from outside the cluster entirely?

### Allowing Traffic from Outside the Cluster

A backup server outside the cluster is not a Pod, so `podSelector` and `namespaceSelector` cannot identify it. If its IP address is known (`192.168.5.10`), use `ipBlock` with a CIDR range. Add it as a separate list entry with its own leading dash, making it an OR rule alongside the API pod entry.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
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
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: prod
        - ipBlock:
            cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 3306
```

The three selector types available in the `from` block (and `to` for egress rules) are:

- `podSelector`: selects pods by their labels.
- `namespaceSelector`: selects namespaces by their labels.
- `ipBlock`: selects external IP address ranges.

The way these selectors are combined **AND** or **OR** depends entirely on how they are written in the manifest YAML file.

### How selectors combine: AND vs. OR

This one is tricky because the logic is controlled entirely by YAML indentation and dashes, not by any keyword. A single dash in the wrong place changes who gets access.

The rule is straightforward once you see it clearly:

- Selectors on the same list entry (with no dash between them) use AND logic, and both must be true.
- Selectors as separate list entries (each with its own dash) use OR logic, and at least one must be true.

#### AND: both conditions are true

The `podSelector` and `namespaceSelector` share one list entry. Traffic is only allowed from a pod labeled `pod: api` that also lives in the `prod` Namespace. The API Pod in `dev` is blocked.

```yaml
from:
  - podSelector:           # one rule entry starts here
      matchLabels:
        pod: api
    namespaceSelector:     # still inside the same entry: AND
      matchLabels:
        kubernetes.io/metadata.name: prod
```

#### OR: one of the conditions is true

Each selector has its own leading dash, making them independent entries. Traffic is allowed from any Pod labeled `pod: api` in any Namespace, OR from any pod in the `prod` Namespace, regardless of its label.

```yaml
from:
  - podSelector:           # first rule entry
      matchLabels:
        pod: api
  - namespaceSelector:     # second rule entry (new dash): OR
      matchLabels:
        kubernetes.io/metadata.name: prod
```

Kubernetes accepts both forms without complaint. The difference only becomes visible when unexpected traffic starts getting through. With the selector logic understood, the last piece is handling outbound traffic from the DB pod.

### Adding an Egress Rule

So far, all rules cover Ingress traffic to the DB pod. Consider a different scenario: an agent inside the DB pod pushes backups to an external server. That traffic from the DB pod is Egress traffic.

Two changes are needed: add `Egress` to `policyTypes`, and add an `egress` section with a `to` field. Since the backup server is external, `ipBlock` is the right selector. The `ports` field specifies port `80` on the backup server.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
spec:
  podSelector:
    matchLabels:
      pod: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - #...
  egress:
    - to:
        - ipBlock:
            cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 80
```

With that egress rule added, the DB pod can push backups out while all other outbound traffic stays blocked.

---

## Complete manifest

Here is the complete manifest with all rules combined.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
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
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: prod
        - ipBlock:
            cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 3306
  egress:
    - to:
        - ipBlock:
            cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 80
```

---

## Summary

A Network Policy binds to Pods via label selectors and defines exactly what traffic is permitted. Ingress rules cover incoming connections, and Egress rules cover outgoing traffic.

Three selector types handle the matching: `podSelector`, `namespaceSelector`, and `ipBlock`.

Selectors in the same list entry use AND logic; selectors in separate entries use OR logic. One misplaced dash is enough to open access to other resources than intended, and Kubernetes will not warn you.

---
