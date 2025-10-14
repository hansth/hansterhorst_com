---
title: Secure K3s with Traefik and Cert-Manager
date: 2025-10-14
tags: ["devops", "k3s", "traefik", "cert-manager", "security"]
draft: false
weight: 3
---

After setting up a secure Ubuntu VPS and installing **K3s** as a lightweight Kubernetes cluster, the next step was to start deploying applications with Kubernetes manifests and make them accessible over HTTPS using a real domain.

In this project, I explain how I secured applications on my K3s cluster using **Traefik** as the Ingress Controller and **cert-manager** to automatically issue and renew SSL certificates from Let’s Encrypt. To demonstrate the setup, I’ll use a simple **Nginx application**.

---

## Pre-requirements

Before starting, make sure you have:

- A registered domain, e.g., `example.dev`.
- A DNS **CNAME** record pointing to your domain, e.g., `example.dev`.
- Public DNS **A/AAAA** records pointing to your VPS IP address.

With these, your domain will correctly route traffic to the K3s cluster.
 
---

## Why Traefik as ingress controller?

Ingress controllers manage external access to Kubernetes services. I could use NGINX or other options, but **Traefik** is the default ingress controller on K3s because:

- It’s lightweight and cloud-native.
- It supports configuration through CRDs.
- It integrates seamlessly with Let’s Encrypt for TLS.

By using Traefik, you get flexible ingress routing without the complexity of other alternatives.

---

## Install Helm for cert-manager

Before we can install cert-manager, we need **Helm**. It's a package manager for Kubernetes, just like `apt` for Ubuntu. With Helm, you install, manage, and update applications in the cluster. It uses preconfigured templates called **Charts** to simplify the deployment process.

**On macOS, install Helm with Homebrew**:
```sh
brew install helm
```

> For other platforms, see the [Helm installation docs](https://helm.sh/docs/intro/install/).

---

## Install cert-manager with Helm

**Cert-manager** is a Kubernetes add-on that automates the management of **TLS/SSL** certificates. It’s widely used in Kubernetes environments to secure communication between applications, services, and external clients.

Without it, you have to manually generate, renew, and distribute certificates, a process that’s both error-prone and time-consuming. Cert-manager simplifies this by handling the entire lifecycle of certificates in a Kubernetes-native way.

**Add the official Jetstack repository**:
```sh
helm repo add jetstack https://charts.jetstack.io --force-update
```
- `helm repo add`: Command to add a new chart repository
- `jetstack`: The local name of this repository.
- `https://charts.jetstack.io`: The URL where the Helm charts are hosted.
- `--force-update`: Forces an update if a repository with this name already exists.

**Update your Helm repositories**:
```sh
helm repo update
```
This will update all the repos.

By default, Helm installs the **Chart** predefined settings. But you can **customize** the deployments by using a `values.yaml` file. This YAML file lets you configure specific parameters for the Helm chart installations without modifying the chart templates.

**Create a `cert-manager-helm-values.yaml` file** for cert-manager:
```yaml
crds:
  enabled: true
```
This enables the CRDs cert-manager needs, such as `Certificate`, `Issuer`, and `ClusterIssuer`, allowing Kubernetes to recognize cert-manager's resources.

> See the Helm Chart on [ArtifactHub](https://artifacthub.io/packages/helm/cert-manager/cert-manager?modal=values).

**Install cert-manager into the cluster**:
```sh
helm install cert-manager jetstack/cert-manager \
  --values cert-manager-helm-values.yaml \
  --namespace cert-manager \
  --create-namespace
```
- `--values`: Specifies the custom Helm values file.
- `--namespace`: Installs cert-manager into the `cert-manager` Namespace.
- `--create-namespace`: Creates the namespace if it doesn’t exist.

**Verify the pods are running**:
```sh
kubectl -n cert-manager get pods
```

Cert-manager should now be running and ready to handle certificate requests.

**OPTIONAL install cert-manager with `kubectl`**:
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.crds.yaml
```

---

## Using Environment Variables in K3s

To keep Kubernetes manifests modular, reusable, and secure, this project uses **environment variables** to inject configuration values during deployment.

Sensitive data, such as credentials, hostnames, and local file paths, are stored in a local `.env` file. The manifests reference these variables using placeholders like `${ROOT_DOMAIN}`, `${WWW_ROOT_DOMAIN}`, or `${YOUR_EMAIL}`.

Example deployment command:

```bash  
envsubst < MANIFEST_FILENAME.yaml | kubectl apply -f -  
```

This approach keeps your manifests **portable**, **environment-agnostic**, and free from **hardcoded values**.

> Read more: [Environment Variables in K8s Manifests](https://hansterhorst.com/devops/k3s-environment-variables)


## Create the ClusterIssuer resource

A **ClusterIssuer** is a cluster-wide Kubernetes resource provided by cert-manager that defines **where and how** to request certificates.

### Staging vs. Production

Let’s Encrypt provides two environments:

- **Staging**: For testing configurations. Certificates are untrusted, but rate limits are generous.
- **Production**: For real, trusted certificates. Rate limits are stricter (50 per domain per week).

Start with **staging** to validate the configuration, then switch to **production** once everything works.

### Staging ClusterIssuer

**Create `cluster-issuer-staging.yaml` manifest**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: "${YOUR_EMAIL}"
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - selector: {}
        http01:
          ingress:
            class: traefik
```

**Apply the manifest file to the cluster**:
```sh
envsubst < cluster-issuer-staging.yaml | kubectl apply -f -
```
This will redirect the manifest file to `envsubst`, which replaces the placeholders with actual values from the `.env` file and pipes the output to `kubectl` for application to the cluster.

**Verify the resources in the cluster**:
```sh
kubectl get clusterissuer
```
```
NAME                     READY   AGE
letsencrypt-staging      True    35s
```
You should see the ClusterIssuers with `READY=True`.

**Show details about the ClusterIssuer**:
```sh
kubectl describe clusterissuer letsencrypt-staging
```
This will show detailed information, like the status.

### Production ClusterIssuer

Create `cluster-issuer-production.yaml` similarly, but with the production ACME server.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "${YOUR_EMAIL}"
    privateKeySecretRef:
      name: letsencrypt-production-account-key
    solvers:
      - selector: {}
        http01:
          ingress:
            class: traefik
```

Don’t apply it until you’re confident the staging configuration works.

---

## Create a Certificate resource

With the ClusterIssuer ready, the next step is to define a **Certificate** resource. This is a custom Kubernetes resource that tells cert-manager which domain names to secure.

**First, create a dedicated `nginx` namespace**:
```bash
kubectl create namespace nginx
```

**Create a `nginx-certificate.yaml` Certificate manifest**:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nginx-certificate
  namespace: nginx
spec:
  secretName: nginx-certificate
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  privateKey:
    rotationPolicy: Always
  dnsNames:
    - "${ROOT_DOMAIN}"
    - "${WWW_ROOT_DOMAIN}"
```

**Apply the manifest file to the cluster**:
```sh
envsubst < nginx-certificate.yaml | kubectl apply -f -
```

**Verify the Certificate**:
```sh
 kubectl -n nginx get certificate
```
You should see the certificate in a `READY=True` state.

---

## Create a test application

To validate the setup, I’ll deploy a **NGINX** test application. Nginx serves a default HTML page, which makes it perfect for testing SSL certificates and ingress routing.

**Create a Deployment using a default image**:
```bash
kubectl -n nginx create deployment nginx-deploy --image=nginx:alpine --port=80
```

**Expose the deployment to access in the browser**:
```bash
kubectl -n nginx expose deployment nginx-deploy \
  --port=80 \
  --target-port=80 \
  --name=nginx-service
```

**Verify the applied resources**:
```bash
kubectl --namespace=nginx get deployment
kubectl --namespace=nginx get service
```
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           5m34s

NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
nginx-service   LoadBalancer   10.43.194.129   <pending>     80:31007/TCP
```

Now Nginx is ready and waiting for an ingress route.

---

## Create Traefik ingress controller

An **IngressRoute** is a Traefik-specific Custom Resource Definition (CRD) that extends the standard Kubernetes Ingress. While a regular Ingress uses generic rules to map traffic to services, an IngressRoute gives you **fine-grained control over routing behavior** using Traefik’s advanced features.

**Create a `ingress-route.yaml` Traefik IngressRoute manifest**:
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-ingressroute
  namespace: nginx
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`${ROOT_DOMAIN}`) || Host(`${WWW_ROOT_DOMAIN}`)
      kind: Rule
      services:
        - name: nginx-service
          port: 80
  tls:
    secretName: nginx-certificate
```

**Apply the manifest file to the cluster**:
```sh
envsubst < nginx-ingressroute.yaml | kubectl apply -f -
```

**Verify the IngressRoute**:
```sh
kubectl -n nginx get ingressroute nginx-ingressroute
```
```
NAME                  READY   SECRET                AGE
nginx-certificate     True    nginx-certificate     1h
```

Visit `https://example.dev` in the browser, you should see the Nginx welcome page with a Let’s Encrypt **staging** certificate. This confirms cert-manager and Traefik are working correctly.

---

## Switch to cert-manager production

Once staging works, it’s time to move to production for real, browser-trusted certificates.

**Apply the production ClusterIssuer**:
```sh
envsubst < cluster-issuer-production.yaml | kubectl apply -f -
```

**Verify the resources in the cluster**:
```sh
kubectl get clusterissuer
```
```
NAME                     READY   AGE
letsencrypt-production   True    12s
letsencrypt-staging      True    93m
```

> If you don’t see the certificate in a `READY=True` state after a few minutes, see **Troubleshooting Certificate Issues**.


Edit the certificate resource to reference `letsencrypt-production` instead of staging, then reapply it.

```yaml
apiVersion: cert-manager.io/v1  
kind: Certificate  
#... 
  issuerRef:  
    name: letsencrypt-production #<---
    kind: #...
```

**Reapply the manifest file to the cluster**:
```sh
envsubst < nginx-certificate.yaml | kubectl apply -f -
```

**Verify certificate details**:
```sh
kubectl -n nginx describe certificate nginx-certificate
```

Visit `https://example.dev` again, you should have a valid Let’s Encrypt **production** certificate for your domain, with automatic renewal configured.

---

## Troubleshooting Certificate Issues

If the certificate isn’t showing `READY=True`, you can inspect cert-manager’s resources to find the cause.

**Check certificate status**:
```bash
kubectl -n nginx describe certificate nginx-certificate
```

**View related Order and Challenge resources**:
```bash
kubectl -n nginx get order,challenge
```

**Inspect details**:
```bash
kubectl -n nginx describe order ORDER_NAME
kubectl -n nginx describe challenge CHALLENGE_NAME
```

**Check cert-manager logs**:
```bash
kubectl -n cert-manager logs -l app.kubernetes.io/instance=cert-manager --tail=50 -f
```

Common issues include incorrect DNS records, HTTP challenges not routing through Traefik, or a wrong ingress class (`traefik`). Once fixed, cert-manager automatically retries the challenge and issues the certificate.

---

## Conclusion

Securing my K3s cluster with **Traefik** and **cert-manager** turned out to be easy and straightforward:

- **Traefik** provides flexible, lightweight ingress routing.
- **Cert-manager** automates SSL issuance and renewal, removing manual overhead.

Applications deployed on the cluster, such as the Nginx test app, are now securely available over HTTPS with trusted Let’s Encrypt certificates. From here, I can deploy other applications like Nexus, Jenkins, or services.

## GitLab

> [**K3S Setup**](https://gitlab.com/devops8614042/k3s-setup)