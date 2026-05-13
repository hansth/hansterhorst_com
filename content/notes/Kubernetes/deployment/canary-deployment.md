---
title: Canary Deployment
date: 2026-03-17
weight: 6
tags: [Deployment Strategy, Deployment]
---
# Canary Deployment

In a Canary deployment strategy, the new version of the application is deployed alongside the current version, with only a small percentage of traffic routed to it. Once the new version has been tested and validated under production conditions, the primary deployment is upgraded to the new version via a rolling update, and the canary deployment is removed.

---

## How It Works

### 1. Deploy the primary version

Deploy the current version as a Deployment named `primary-deploy`, with five replicas and the label `version: v1`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: primary-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: frontend-app
      version: v1
  template:
    metadata:
      labels:
        app: frontend-app
        version: v1
    spec:
      containers:
        - name: primary-container
          image: frontend-app:1.0
```

### 2. Create a Service

The Service uses the `app: frontend-app` label as its selector, routing all traffic to the primary Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend-app
  ports:
    - port: 80
      targetPort: 8080
```

### 3. Deploy the canary version

Deploy the new version as a separate Deployment named `canary-deploy`. The image is updated to version 2.0, and the label is set to `version: v2`. The replica count is set to `1` to limit the percentage of traffic it receives.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend-app
      version: v2
  template:
    metadata:
      labels:
        app: frontend-app
        version: v2
    spec:
      containers:
        - name: canary-container
          image: frontend-app:2.0
```

Because both Deployments share the same label `app: frontend-app`, the Service routes traffic to Pods in both. With five Pods in the primary and one in the canary, the traffic split is approximately 83% to the primary and 17% to the canary.

### 4. Promote the primary deployment

Once the canary Deployment has been tested and validated, update the primary Deployment's container image to version 2.0, then delete the canary Deployment if the rolling update was successful. Because `selector.matchLabels` is immutable after creation, only the Pod template can be updated.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: primary-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: frontend-app
      version: v1
  template:
    metadata:
      labels:
        app: frontend-app
        version: v1
    spec:
      containers:
        - name: primary-container
          image: frontend-app:2.0
```

---

## Limitations of the Native Approach

In this setup, traffic distribution depends entirely on the relative number of Pods in each Deployment. For example, routing 1% of traffic to a canary deployment would require at least 100 Pods overall.

If you need precise, replica‑independent traffic control, a service mesh such as Istio can specify exact traffic percentages per Deployment, even when each Deployment runs only a single Pod.

---
