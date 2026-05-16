---
title: Service Account
date: 2026-03-14
weight: 11
tags: [Service Account, Security]
---
# Service Accounts

A Service Account is a non-human Kubernetes account that provides a unique identity within a cluster. Pods, system components, and external entities use a Service Account's credentials to authenticate against the Kubernetes API and to have security policies enforced against them.

Each Service Account is associated with a token that authenticates it to the Kubernetes API. If a Service Account is an ID card, the token is the barcode on that card. Kubernetes reads the token to verify the Service Account's identity, just as a scanner reads a barcode to verify a cardholder.

---

## Two Types of Accounts

Kubernetes recognizes two account types:

- **User accounts**: used by **users**, such as administrators performing cluster operations or developers deploying applications.
- **Service accounts**: used by **applications** that need to interact with the Kubernetes API. Prometheus uses a Service Account to query the API for performance metrics. Jenkins uses one to deploy applications to the cluster.

---

## Default Service Account

When a Kubernetes cluster is set up, it creates a Service Account named `default` in every namespace:

```bash
kubectl get serviceaccount
```

```
NAME      SECRETS   AGE
default   0         4d1h
```

Whenever a Pod is created without specifying a Service Account, Kubernetes automatically attaches the default Service Account. This is visible in the Pod description:

```yaml
Name:             web-pod
Namespace:        default
Service Account:  default
```

The Service Account token is mounted as a projected volume inside the Pod at:

```
/var/run/secrets/kubernetes.io/serviceaccount
```

```bash
kubectl exec web-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount
# ca.crt  namespace  token
```

The directory contains three files: `ca.crt`, `namespace`, and `token`. The `token` file contains the Bearer token used for API authentication.

> **Note**: The default Service Account has very limited permissions. For most use cases, create a dedicated Service Account with the appropriate RBAC permissions.

---

## Creating a Service Account

**Imperatively**
```bash
kubectl create serviceaccount custom-sa
```

**Declaratively**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: custom-sa
```

**Inspect a Service Account**
```bash
kubectl describe serviceaccount custom-sa
```
```
Name:                custom-sa
Namespace:           labs
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```

A freshly created Service Account starts with most properties set to `none`.

---

## Associating a Service Account with a Pod

To use a specific Service Account in a Pod, add the `serviceAccountName` field to the Pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-pod
spec:
  serviceAccountName: custom-sa
  containers:
  - image: nginx
    name: sa-container
```

> **Important**: The Service Account of an existing Pod cannot be updated. The Pod must be deleted and recreated. For Deployments, updating the Service Account in the Pod template triggers an automatic rollout.

Kubernetes automatically creates a short-lived token, mounts it as a projected volume, rotates it while the Pod is running, and expires it when the Pod is deleted:

```bash
kubectl exec sa-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount
# ca.crt  namespace  token
```

---

## Disabling Automatic Token Mounting

To prevent Kubernetes from automatically mounting a Service Account token, set `automountServiceAccountToken` to `false`.

At the Service Account level, this applies to all Pods using that Service Account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: custom-sa
automountServiceAccountToken: false
```

At the Pod level, this overrides the Service Account setting for that specific Pod:

```yaml
spec:
  serviceAccountName: custom-sa
  automountServiceAccountToken: false
```

---

## Tokens for External Use

For Pods running inside the cluster, Kubernetes automatically creates and mounts a token. For external use, such as configuring a CI/CD pipeline, connecting an external monitoring tool, or authenticating a dashboard, a token must be created manually:

```bash
kubectl create token custom-sa
# eyJhbGciOiJSUz...-m02TRpbA
```

This creates a short-lived ServiceAccount token (a JWT) by sending a `TokenRequest` to the Kubernetes API server, which returns a signed JWT bound to that Service Account. The token is printed to stdout. By default, it is valid for one hour. The duration can be extended with the `--duration` flag:

```bash
kubectl create token custom-sa --duration=48h
```

The token is used as a Bearer token in API requests:

```bash
curl https://<kubernetes-api-server>/api/v1/pods \
  -H "Authorization: Bearer eyJhbGciOiJSUz...-m02TRpbA"
```

### Decoding a Token

The token payload can be inspected using `jq`:

```bash
jq -R 'split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson' <<< eyJhbGciOiJSUz...-m02TRpbA
```
- `-R`: reads the input as a raw string, not JSON.
- `split(".")`: splits the JWT on dots into its three parts: header, payload, and signature.
- `select(length > 0)`: filters out empty strings.
- `.[0],.[1]`: selects the header and payload, skipping the signature.
- `@base64d`: base64-decodes each part.
- `fromjson`: parses the decoded strings into JSON for pretty printing.
- `<<<`: a bash here-string feeding the raw JWT to `jq`.

Example output:

```json
{
  "alg": "RS256",
  "kid": "tRdkXwroSdFyW4SJrjM_nFtEN2OnUJmR_5NgEnq4tkY"
}
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1773527565,
  "iat": 1773523965,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "jti": "a8e8b0af-1e6e-4b7d-af6c-6b80f09b7b57",
  "kubernetes.io": {
    "namespace": "labs",
    "serviceaccount": {
      "name": "custom-sa",
      "uid": "affd5302-ff4d-4750-a13f-4460e69615c6"
    }
  },
  "nbf": 1773523965,
  "sub": "system:serviceaccount:labs:custom-sa"
}
```
- `iss` / `aud`: The API server issued the token, and it is intended for the API server.
- `iat` / `exp`: Issued at a specific time with a 1-hour TTL (`exp - iat = 3600`).
- `sub`: The identity Kubernetes RBAC evaluates when this token is presented.
- `jti`: A unique token ID, enabling revocation.
- `nbf`: Not-before time, same as `iat`.

The token can also be decoded at [jwt.io](https://jwt.io/).

---

## Token Changes in Kubernetes 1.22 and 1.24

### Before 1.22

When a Service Account was created, Kubernetes automatically generated a secret containing a JWT token. This token had no expiry date, was not bound to any audience or object, and required a separate secret object per Service Account.

### Kubernetes 1.22

The Token Request API was introduced to address these issues. Tokens generated through this API are audience-bound, time-bound, and tied to a specific Pod. When a Pod is created, the Service Account admission controller generates a short-lived token through the Token Request API and mounts it as a projected volume. The projected volume automatically refreshes the token before it expires.

### Kubernetes 1.24

Automatic secret generation was removed entirely. When a Service Account is created, Kubernetes no longer generates a corresponding secret with a token.

If a non-expiring token stored in a secret is required in environments where the Token Request API is not available, one can be created manually:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: custom-sa-token
  annotations:
    kubernetes.io/service-account.name: custom-sa
type: kubernetes.io/service-account-token
```

> **Important**: Create the Service Account before creating the secret, otherwise the secret will not be populated with a token. Non-expiring tokens should only be created when the Token Request API is not an option and the security trade-off is explicitly accepted.

---

## Token Version Overview

| Feature                 | Before 1.22               | 1.22+                                  | 1.24+                  |
|-------------------------|---------------------------|----------------------------------------|------------------------|
| Token type              | Static secret, no expiry  | Projected volume via Token Request API | Same as 1.22           |
| Auto-created secret     | Yes                       | Yes                                    | No                     |
| Token expiry            | None                      | Time-bound, auto-refreshed             | Time-bound             |
| Manual token generation | `kubectl describe secret` | `kubectl describe secret`              | `kubectl create token` |
---
