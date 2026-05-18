---
title: Webhook Admission Controllers
date: 2026-05-17
weigh: 2
tags: [Controllers, Webhook]
---
# Webhook Admission Controllers

An Admission Controller is a component that intercepts requests to the Kubernetes API server after authentication and authorization, but before the resource is stored in `etcd`.

---

## The Request Flow

Every `kubectl` request passes through the `kube-apiserver`, where it is authenticated using certificates, authorized via RBAC, and then processed by Admission Controllers before the object is created and stored in `etcd`.

```
           ┌─────────────────────────────────────────────────────┐
           │ kube-apiserver                                      │
┌───────┐  │  ┌──────────────┐   ┌─────────────┐   ┌───────────┐ │  ┌───┐
│kubectl│──┼─▶│Authentication│──▶│Authorization│──▶│ Admission │─┼─▶│Pod│
└───────┘  │  │ certificates │   │    RBAC     │   │Controllers│ │  └───┘
           │  └──────────────┘   └─────────────┘   └───────────┘ │
           └─────────────────────────────────────────────────────┘
```

---

## The Limits of RBAC

RBAC determines which API operations a user is allowed to perform. However, there are scenarios where fine-grained control over incoming requests is required:

- Rejecting Pod creation requests that reference images from public registries, allowing only images from a specific internal registry.
- Enforce that the `latest` image tag is never used.
- Rejecting requests to run containers as the root user.
- Permitting only certain Linux capabilities.
- Ensuring that pod metadata always includes specific labels.

None of these can be configured with RBAC alone. Admission Controllers solve this gap.

---

## What Admission Controllers Do

Admission Controllers enforce security measures and control how a cluster is used. Beyond validating incoming requests, an Admission Controller can also mutate the request itself or perform additional backend operations before the object is created.

| Type       | Behaviour                                          |
|------------|----------------------------------------------------|
| Mutating   | Modifies the request object before it is processed |
| Validating | Inspects the request and allows or denies it       |

Some controllers do both. The namespace lifecycle admission controller is a validating example: it inspects a request and allows or denies it without modification. The default storage class admission controller is a mutating example: when a PVC creation request arrives without a storage class specified, it attaches the cluster's default storage class before the object is created.

---

## Order of Invocation

Mutating admission controllers are always invoked before validating ones. Any change a mutating controller makes must be visible to the validating controllers that follow.

Consider two controllers: `NamespaceAutoProvisioning` (mutating, creates a namespace if it does not exist) and `NamespaceExists` (validating, rejects requests targeting a non-existent namespace). If the validating controller ran first, it would reject every request for a namespace not yet created. Running the mutating controller first ensures the namespace exists before validation occurs.

When any admission controller in the chain rejects a request, the entire request is rejected and an error is returned.

---

## Admission Webhooks

Built-in admission controllers are compiled into Kubernetes. Two special plugins extend this mechanism with custom logic:

- **MutatingAdmissionWebhook**
- **ValidatingAdmissionWebhook**

After a request passes through all built-in admission controllers, it reaches the configured webhook. The webhook calls an external admission webhook server, passing it an **AdmissionReview** object in JSON format containing the requesting user, operation type, target object, and object details.

The server responds with its own AdmissionReview object. If the `allowed` field is `true`, the request proceeds. If `false`, it is rejected.

---

## Webhook Setup

### Step 1: Deploy the Webhook Server

The webhook server is an API server that must accept requests on `validate` and `mutate` endpoints and respond with the AdmissionReview JSON format Kubernetes expects. It can be written in any language.

Example handlers:

- **validate**: Receives the request, compares the object name with the submitting username, and rejects the request if they match.
- **mutate**: Reads the username and responds with a JSON patch operation that adds it as a label on the object.

A JSON patch operation is a list of patch objects, each specifying an operation type (`add`, `remove`, `replace`, `move`, `copy`, or `test`), a path within the JSON object (for example `/metadata/labels/user`), and a value for `add` operations. The patch is returned as a base64-encoded string.

### Step 2: Host the Server

Two options:

- Run it as a standalone external server.
- Containerize it and deploy it as a Deployment inside the cluster, with a Service (for example `webhook-service`) to expose it.

### Step 3: Create the Webhook Configuration Object

**Create the configuration**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: my-webhook
webhooks:
  - name: example.com
    clientConfig:
      service:
        namespace: default
        name: webhook-service
      caBundle: <base64-encoded-CA-bundle>
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
```

- `clientConfig` defines how the API server reaches the webhook server. For an external server, provide a URL directly. For an in-cluster Service, provide the namespace and name. Communication must be over TLS; a certificate pair is required on the server, and the CA bundle must be provided in `caBundle`.
- `rules` define when the webhook is invoked. Use `apiGroups`, `apiVersions`, `operations`, and `resources` to narrow the scope. The example above triggers only on Pod creation requests.

Once applied, every matching request calls the webhook server and is allowed or rejected based on its response.