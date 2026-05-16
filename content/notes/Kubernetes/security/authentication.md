---
title: Authentication
date: 2026-05-10
weight: 2
tags: [Security, Authentication]
---
# Authentication

A Kubernetes cluster consists of multiple nodes, physical or virtual, along with various components that work together. Several types of actors interact with a cluster:

- **Administrators** who access the cluster to perform administrative tasks.
- **Developers** who access the cluster to test or deploy applications.
- **End users** who access the applications deployed on the cluster.
- **Third-party applications** that access the cluster for integration purposes.

---

## Who Needs to Authenticate?

End users who access applications deployed on the cluster are managed internally by the applications themselves and are not covered here.

The focus is on users accessing the Kubernetes cluster for administrative purposes, which leaves two categories:

- **Users**: administrators and developers.
- **Services**: other processes, services, or applications that require cluster access.

### User Accounts vs. Service Accounts

Kubernetes does not manage User Accounts natively. It relies on external sources, such as a file containing user details, certificates, or a third-party identity service like LDAP, to manage these users.

You cannot create users in a Kubernetes cluster or list them the way you would other resources.

Service accounts are different. Kubernetes can create and manage service accounts natively through the Kubernetes API.

---

## The Role of the API Server

All user access is managed by the API server. Whether you access the cluster via `kubectl` or directly via the API, every request passes through the `kube-apiserver`. The API server authenticates each request before processing it.

### Supported Authentication Mechanisms

The `kube-apiserver` supports several authentication mechanisms:

- **Certificates**: TLS client certificate authentication.
- **Third-party authentication protocols**: such as LDAP, Kerberos, and others.
- **Service accounts**: for programmatic access by internal workloads.

> **Note**: Static password files (`--basic-auth-file`) and static token files (`--token-auth-file`) were deprecated in Kubernetes v1.16 and removed in v1.19. They are documented below for historical reference only and must not be used in any current or production cluster.

---

## Static Password Files (Removed)

A CSV file containing user credentials with three columns: **password**, **username**, and **user ID**. A fourth column can optionally assign users to groups.

```csv
password123,admin,u0001
password456,developer,u0002
```

The file was passed to the `kube-apiserver` using the `--basic-auth-file` flag:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --basic-auth-file=<LOCATION_USERS_CSV_FILE>
```

Credentials were passed in `curl` requests as follows:

```bash
curl -v -k https://192.168.3.110:6443/api/v1/pods \
  -u "admin:password123"
```

---

## Static Token Files (Removed)

A CSV file containing user credentials with three columns: **token**, **username**, and **user ID**. A fourth column can optionally assign users to groups.

```csv
KpjCvBI7rCFAHYPKByTlzRb7gulcUc4B,user10,u0010
rJjncHmvtXHc6M1WQddhtvNyhgTdxSC,user11,u0011
```

The file was passed to the `kube-apiserver` using the `--token-auth-file` flag:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --token-auth-file=<LOCATION_USERS_CSV_FILE>
```

Tokens were passed as an Authorization Bearer Token in `curl` requests:

```bash
curl -v -k https://192.168.3.110:6443/api/v1/pods \
  --header "Authorization: Bearer KpjCvBI7rCFAHYPKByTlzRb7gulcUc4B"
```

Both mechanisms stored credentials in plain text, making them inherently insecure.

---

## Service Accounts

Service accounts are used by applications that need to interact with the Kubernetes API, such as monitoring tools like Prometheus querying the API for metrics, or CI/CD tools like Jenkins deploying applications to the cluster.

### Creating and Using a Service Account

**Create a service account:*
```bash
kubectl create serviceaccount dashboard-sa
```

**List all service accounts:*
```bash
kubectl get serviceaccounts
```

When a service account is created, Kubernetes also creates a corresponding secret containing a token. This token is used by external applications to authenticate against the Kubernetes API.

**Inspect the token:*
```bash
kubectl describe secret dashboard-sa-token-kbpppdm
```

The token can be passed as a Bearer token in API requests:

```bash
curl https://<kubernetes-api>/api/v1/pods \
  --header "Authorization: Bearer <token>"
```

> **Note**: Assigning the correct permissions to a service account is done through RBAC.

---

## Conclusion

Authentication in Kubernetes separates Users, managed through external sources, from Service Accounts, managed natively through the Kubernetes API.

All requests pass through the `kube-apiserver`, which authenticates each one before processing. Current supported mechanisms are certificates, third-party protocols such as LDAP, and service accounts.

Applications running inside the cluster authenticate using service account tokens, mounted automatically as a projected volume into the pod.
