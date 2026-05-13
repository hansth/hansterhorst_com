---
title: Blue-Green Deployment
date: 2026-03-17
weight: 5
tags: [Deployment Strategy, Deployment]
---

# Blue-Green Deployment

Blue-Green is a deployment strategy where a new version of an application runs alongside the current version. The current version is called **Blue**, and the new version is called **Green**. Traffic is initially routed to the Blue version while the Green version is tested. Once all tests pass, traffic will be routed entirely to the Green version.
  
---  

## How It Works

### 1. Deploy the Blue version

First, you deploy the current version of the application. The Deployment is called `blue-deploy`, with five replicas and the label `version: v1`.

```yaml  
apiVersion: apps/v1  
kind: Deployment
metadata:
  name: blue-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: blue-green-app
      version: v1
  template:
    metadata:
      labels:
        app: blue-green-app
        version: v1
    spec:
      containers:
        - name: blue-container
          image: blue-green-app:1.0
```  

### 2. Create a Service pointing to the Blue version

The Service uses the `version: v1` label as the `selector`, which routes all traffic to the Blue Pods.

```yaml  
apiVersion: v1
kind: Service
metadata:
  name: blue-green-svc
spec:
  selector:
    app: blue-green-app
    version: v1
  ports:
    - port: 80
      targetPort: 8080
```  


### 3. Deploy the Green version

Deploy the new version as a separate Deployment named `green-deploy`. The image is updated to a newer version, and the label is set to `version: v2`. Traffic is not yet routed to this.

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: blue-green-app
      version: v2
  template:
    metadata:
      labels:
        app: blue-green-app
        version: v2
    spec:
      containers:
        - name: green-container
          image: blue-green-app:2.0
```  

### 4. Switch traffic to the Green version

Once all tests pass, update the Service `selector` from `v1` to `v2`, and traffic will be immediately routed to the Green Pods.

```yaml  
apiVersion: v1
kind: Service
metadata:
  name: blue-green-svc
spec:
  selector:
    app: blue-green-app
    version: v2
  ports:
    - port: 80
      targetPort: 8080
```  

All traffic is now routed to Pods labeled `version: v2`. The Blue Deployment remains available and can be used to roll back by reverting the Service selector to `v1`.

---
