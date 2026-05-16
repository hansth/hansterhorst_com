---
title: Authorization
date: 2026-05-13
weight: 7
tags: [Security, Authorization]
---
# Authorization

Authentication determines identity. Authorization determines what an authenticated user or process is allowed to do.

Once a user or machine process gains access to a Kubernetes cluster, the set of permitted operations is governed entirely by the authorization layer. Different actors require different levels of access:

- **Developers** should be able to deploy applications but not modify cluster-level configuration, such as adding or removing nodes or changing storage and networking settings.
- **External applications**, such as monitoring agents, should be granted only the minimum permissions required for their specific function.
- **Teams sharing a cluster** via namespaces should be restricted to their own namespace and not interact with resources belonging to other teams.

Authorization enforces these boundaries.

---

## Authorization Mechanisms

Kubernetes supports several authorization mechanisms, each representing a different approach to deciding whether a request should be permitted.

### Node Authorization

The `kube-apiserver` is accessed not only by human users but also by the `kubelet` running on each node. `kubelet` reads information about Services, Endpoints, Nodes, and Pods, and reports node status back to the API server. These internal communications are handled by the **Node Authorizer**, a dedicated authorizer built for this purpose.

For the Node Authorizer to recognize a kubelet, the kubelet must belong to the `system:nodes` group, and its username must be prefixed with `system:node`. Any request matching this profile is granted the privileges required for kubelet operation, keeping internal cluster communication on a separate, well-defined authorization track.

### Attribute-Based Access Control (ABAC)

ABAC associates a user or group directly with a set of permissions defined in a JSON policy file. The policy file is passed to the `kube-apiserver` at startup, and every user or group requires its own policy entry.

The fundamental problem with ABAC is operational: whenever access needs to change, the policy file must be manually edited, and the `kube-apiserver` restarted. Because there is no live-update path, ABAC is difficult to maintain at scale and is generally not recommended for active clusters.

### Role-Based Access Control (RBAC)

RBAC addresses ABAC's maintainability problems by introducing an intermediary layer: **Roles**.

Instead of binding permissions directly to users, a Role is defined with a specific set of permissions, for example, a `developer` Role that allows viewing and managing Pods in a given namespace. Users are then bound to that Role. When permissions need to change, the Role itself is updated, and the change takes effect immediately for every user bound to it.

RBAC is the standard and recommended authorization mechanism in Kubernetes.

### Webhook Authorization

Webhook authorization delegates authorization decisions entirely to an external service. When a request arrives, Kubernetes makes an API call to the configured external service, passing the user's identity and the requested action. The external service responds with an allowed or deny decision, which Kubernetes enforces.

Open Policy Agent (OPA) is a common example. This approach is useful when an organization wants to centralize authorization policy across multiple systems using a single external engine, or when authorization logic is too complex for built-in mechanisms.

### AlwaysAllow and AlwaysDeny

Two additional modes exist for specific circumstances:

- **AlwaysAllow**: permits every request without performing any authorization checks. The default if no authorization mode is explicitly configured.
- **AlwaysDeny**: rejects every request unconditionally.

These modes are rarely appropriate for production use but may be used in specific testing or lockdown scenarios.

---

## Configuring Authorization Modes

Authorization modes are configured via the `--authorization-mode` flag on the `kube-apiserver`. Multiple modes can be active simultaneously using a comma-separated list:

```
--authorization-mode=Node,RBAC,Webhook
```

### The Authorization Chain

When multiple modes are configured, incoming requests are evaluated against each authorizer in the order listed:

1. A request arrives and is passed to the first authorizer in the list.
2. If the authorizer denies the request or abstains, the request moves to the next authorizer.
3. If any authorizer approves the request, evaluation stops, and the request is permitted.
4. If all authorizers deny the request, it is rejected.

**Example** using `Node,RBAC,Webhook`:

1. A developer sends a request to the API server.
2. The Node Authorizer evaluates it. The request is not from a kubelet, so it denies it and passes it along.
3. The RBAC authorizer evaluates it. The developer's Role grants the necessary permission, so the request is approved.
4. Evaluation stops. The developer is granted access.

The order of modes is meaningful. A single approval anywhere in the chain is enough to grant access. A denial only moves the request to the next authorizer.

---

## Mechanism Overview

| Mechanism   | Description                                            | Management Complexity                             |
|-------------|--------------------------------------------------------|---------------------------------------------------|
| Node        | Handles kubelet requests to the API server             | Automatic                                         |
| ABAC        | Policy file maps users/groups to permissions           | High: requires file edits and API server restarts |
| RBAC        | Permissions defined in Roles; users bound to Roles     | Low: changes apply immediately                    |
| Webhook     | Delegates decisions to an external service such as OPA | Depends on external system                        |
| AlwaysAllow | Permits all requests without checks                    | N/A                                               |
| AlwaysDeny  | Denies all requests                                    | N/A                                               |

---