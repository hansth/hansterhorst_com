---
title: Deploy Nexus Repository on K3s
date: 2025-10-30
tags: ["devops", "nexus", "k3s"]
draft: false
weight: 6
---

# Deploy Nexus Repository on a K3s Cluster

After completing the **TWN DevOps Bootcamp**, I wanted to recap my learning by creating a series of **portfolio projects** that combine my new DevOps skills with my professional experience in software development.

Here’s the technology stack I’ve been working with:

- **Backend**: Java, Spring Boot, MySQL, PostgreSQL, OpenAPI, Liquibase
- **Frontend**: Angular, React
- **CI/CD & Tools**: Jenkins, Docker, Git

In the [previous project](https://gitlab.com/devops8614042/build-tools), I used build tools like **Maven**, **Gradle**, **NPM**, and **Docker** to generate various types of **artifacts**. In this project, I’ll explain the steps I took to install and configure **Nexus Repository Manager** on a single-node **K3s cluster** running on an **Ubuntu VPS**.

---

## Why Use Nexus Repository?

In modern software development, managing the **artifacts** generated during the build and packaging process is essential. To ensure consistency, reliability, and collaboration, these artifacts must be **stored**, **versioned**, and **shared** across teams and **CI/CD** pipelines.

This is where **artifact repositories** and **repository managers** play an important role.

An **artifact repository** serves as a central hub where all build outputs are stored and versioned. A **repository manager** extends this by adding features like access control, caching, and support for multiple formats (Maven, npm, NuGet, Docker, etc.). It provides a single, unified interface for managing artifacts across projects—making the entire build and release process faster, more consistent, and easier to maintain.

**Sonatype Nexus Repository** is one of the most widely adopted and flexible options available. It allows teams to:

- Host internal artifacts securely.
- Proxy public repositories such as [Maven Repository](https://mvnrepository.com/repos/central) and [NPM Register](https://www.npmjs.com) for faster, cached builds.
- Combine multiple repositories into logical groups for simplified access.
- Integrate seamlessly with CI/CD pipelines.
- Manage cleanup policies, backups, and access permissions.

In short, **Nexus acts as a central hub of artifact management**, helping teams **centralize, secure, and streamline** their software delivery processes.

---

## Installing Nexus on a K3s Cluster

In this project, I’ll deploy **Sonatype Nexus Repository** on K3s using Kubernetes manifests. The deployment will:

- Run as a **StatefulSet** to ensure persistent storage.
- Expose Nexus through an **IngressRoute** resource secured with **TLS certificates** from Cert-Manager.
- Follow **production-ready** best practices.

By the end, I’ll have a fully functional Nexus installation, ready to host repositories for **Maven**, **npm**, and **Docker**.

---

## Prerequisites

Before proceeding, ensure you have the following:

- A running **K3s** cluster.
- `kubectl` is installed and configured locally.
- **Traefik** is set up as the default Ingress controller.
- **Cert-Manager** installed, with a **ClusterIssuer** configured for automatic TLS certificate issuance.
- A registered **domain name** (e.g., `example.dev`).
- DNS **CNAME** records (`nexus`, `docker`) pointing to your domain.

---

## Using Environment Variables in K3s

To keep Kubernetes manifests modular, reusable, and secure, this project uses **environment variables** to inject configuration values during deployment.

Sensitive data, such as credentials, hostnames, and local file paths, are stored in a local `.env` file. The manifests reference these variables using placeholders like `${K3S_LOCAL_STORAGE}`, `${NODE_HOSTNAME}`, or `${ROOT_DOMAIN}`.

Example deployment command:

```bash  
envsubst < MANIFEST_FILENAME.yaml | kubectl apply -f -  
```

This approach keeps your manifests **portable**, **environment-agnostic**, and free from **hardcoded values**.

> Read more: [Environment Variables in Kubernetes Manifests](https://www.hansterhorst.com/devops/k3s-environment-variables/)

---

## Updating the UFW Firewall

Before installing Nexus, configure **UFW (Uncomplicated Firewall)** to open the necessary ports for the Nexus repository and Docker registry.

**1. Log in to your server**:
```bash  
ssh <USERNAME>@<SERVER_IP>  
```

**2. Open the required ports**:
```bash  
# Nexus repository
sudo ufw allow 8081
# Docker registry
sudo ufw allow 5000
```

---

## Create a Namespace

In Kubernetes, a **Namespace** helps organize and isolate resources within a cluster. Think of the cluster as an office building, each Namespace is a separate office with its own staff, resources, and policies, while sharing the same infrastructure.

It’s best practice to create namespaces for DevOps tools such as **Nexus**, **Jenkins**, or **Grafana**, ensuring they remain logically separated from application workloads.

**File**: [`namespace.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/namespace.yaml)

**What it does**:
- Declares a Namespace resource named `nexus`.
- Isolates Nexus from other workloads in the cluster.
- Simplifies resource management, access control, and network policies.

**Apply the manifest**:
```bash  
kubectl apply -f namespace.yaml  
```

---

## Configure Persistent Storage

**Nexus** is a **stateful application**, which means it needs persistent storage to retain repositories, configuration, and metadata across Pod restarts. Kubernetes Pods are **ephemeral**, when they restart or reschedule, any internal data is lost.

To ensure persistent storage, I’ll define three essential Kubernetes resources:

1. **StorageClass (SC)**: Defines how storage is provisioned.
2. **PersistentVolume (PV)**: Represents the actual storage device or location.
3. **PersistentVolumeClaim (PVC)**: Requests and binds to the available storage.

### StorageClass (SC)

A **StorageClass** is a cluster-wide resource that defines how persistent storage is provisioned in a cluster. While K3s provides a default `local-path` class, creating a custom `local-storage` class gives me better control over data location and lifecycle.

**File**: [`local-storageclass.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/local-storageclass.yaml)

**What it does**:

- Creates a `local-storage` class with `no-provisioner`.
- Ensures administrators manually create PersistentVolumes bound to nodes.
- Uses `WaitForFirstConsumer` mode so the volume binds only when a Pod requests it.

**Apply the manifest**:
```bash  
kubectl apply -f local-storageclass.yaml  
```

### PersistentVolumeClaim (PVC)

A **PersistentVolumeClaim (PVC)** is an application’s request for persistent storage. It specifies the desired capacity, access mode, and storage class, while Kubernetes automatically binds it to a matching PersistentVolume.

**File**: [`volumes.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/volumes.yaml)

**What it does**:
- Requests a `10 Gi` volume from the `local-storage` class.
- Ensures the volume can only be mounted, by one node at a time (`ReadWriteOnce`).
- Provides Nexus with reliable, persistent data storage.

### PersistentVolume (PV)

A **PersistentVolume** represents the physical storage available in the cluster, it could be a local directory on the VPS, a Network File System (NFS) share, or a cloud-based storage service like AWS Elastic Block Store (EBS).

In this setup, the PV is configured to store data directly in the VPS’s `/mnt` directory. This approach provides full control over the underlying data while keeping it easily accessible on the VPS. It also streamlines the backup process and simplifies routine maintenance tasks.

**File**: [`volumes.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/volumes.yaml)

**What it does**:

- Reserves `10 Gi` of local disk space for Nexus.
- Uses `nodeAffinity` to ensure the volume binds to a specific node.
- Maps a local directory `/mnt/K3S_LOCAL_STORAGE/nexus` as the storage path.
- Retains data even if the Pod is deleted (`Retain` policy).

>**_Note_**: Replace `K3S_LOCAL_STORAGE` with your actual storage path and `NODE_HOSTNAME` with the hostname of the K3s node.

**Check your node hostname**:
```bash  
kubectl get nodes  
```

Copy/paste the value under the `NAME` column as your `${NODE_HOSTNAME}` in the manifests.

### Prepare the Storage Directory

Before applying the storage manifests, I'll create a local storage directory on the VPS with the correct permissions.

```bash  
ssh <USER>@<SERVER_IP>  
sudo mkdir -p <K3S_LOCAL_STORAGE>/<NEXUS>  
sudo chown 200:200 -R <K3S_LOCAL_STORAGE>/<NEXUS>   
```
- `200:200`: Corresponds to the Nexus container’s UID and GID in the StatefulSet.
- `-R`: Applies ownership recursively.

**Apply the manifest**:
```bash  
envsubst < services.yaml | kubectl apply -f -  
```

**Verify storage status**:
```bash  
kubectl -n nexus get pv  
kubectl -n nexus get pvc  
```

---

## Deploy Nexus as a StatefulSet

A **StatefulSet** manages stateful workloads, ensuring stable network identities and persistent storage. Unlike a **Deployment** (used for stateless applications), a **StatefulSet** is ideal for applications like **Nexus**, **Jenkins**, or databases like **PostgreSQL**.

**File**: [`statefulset.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/statefulset.yaml)

**What this manifest does**:

- Deploys **Nexus** as a **single-replica StatefulSet** (one Pod).
- Mounts a **PersistentVolumeClaim** at `/nexus-data` for durable storage.
- Defines a `securityContext` (`UID/GID 200`) for proper file access.
- Adds **liveness** and **readiness probes** for health monitoring.
- Defines **resource requests and limits** for CPU and memory management.

I divided the YAML file into sections to explain the different areas of this StatefulSet.

### Basic Information

Defines general resource metadata.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nexus-statefulset
  namespace: nexus
```
- `apiVersion`:  API version for StatefulSets.
- `kind`:  The type of Kubernetes resource.
- `metadata.name`:  The StatefulSet’s name.
- `namespace`:  Deploys Nexus within the `nexus` namespace.

---

### StatefulSet Specification

This defines the desired behavior and configuration of the StatefulSet.

```yaml
spec:
  serviceName: nexus-headless
  replicas: 1
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
```
- `serviceName`: Refers to a **headless service** (`nexus-headless`) that provides stable network identities for pods. StatefulSets require this.
- `replicas`: Runs **one** Nexus pod.
- `updateStrategy.type`: The pods are updated one at a time.
- `podManagementPolicy`: Ensures pods are started or updated in sequential order.

---

### Pod Template Metadata

These define how pods are identified and what they look like.

```yaml
  selector:
    matchLabels:
      app: nexus
  template:
    metadata:
      labels:
        app: nexus
```
- The `selector` ensures the StatefulSet manages pods labeled with `app: nexus`.
- The `template` defines the pod spec used for creating these pods.
---

### Pod Security Context

Defines the user and group IDs under which the Nexus container runs.

```yaml
securityContext:
  runAsUser: 200
  runAsGroup: 200
  fsGroup: 200
```
- `runAsUser` / `runAsGroup`: Runs Nexus as a **non-root user** with:
    - `UID:GID 200`: User- and group ID `200.
- `fsGroup`: Grants write access to mounted volumes for that group.

> **_Note_**: Running as a non-root user hardens the container’s security and prevents permission conflicts when writing to persistent storage.

---

### Container Definition

Specifies the container image and runtime details.

```yaml
containers:
  - name: nexus
    image: sonatype/nexus3:3.85.0-java17-alpine
    imagePullPolicy: IfNotPresent
```
- `name`: Container identifier name within the pod.
- `image`: Uses a fixed Nexus 3 LTS image built with Java 17 on Alpine.
- `imagePullPolicy`: Pulls only if the image is not already available locally.

> **_Note_**: Specify a specific image version ensures predictable, stable deployments and easier rollback.

---

### Container Security

Defines security settings at the container level.

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  runAsNonRoot: true
```
- `allowPrivilegeEscalation: false`: Blocks privilege escalation.
- `readOnlyRootFilesystem: false`: Keeps `/nexus-data` writable.
- `runAsNonRoot: true`: Enforces non-root operation.

> **_Note_**: These configurations align with Kubernetes’ best security practices.

---

### Ports

Defines which ports the container exposed.

```yaml
ports:
  - name: nexus
    containerPort: 8081
  - name: docker
    containerPort: 5000
```
- `8081`: Nexus web interface.
- `5000`: Docker registry port.

> **_Note_**: Naming ports helps when defining Services, Ingress rules, and network policies.

---

### Environment Variables

Sets environment variables to configure Java runtime options.

```yaml
env:
  - name: INSTALL4J_ADD_VM_PARAMS
    value: "-Xms1g -Xmx2g -XX:MaxDirectMemorySize=2g -Djava.util.prefs.userRoot=/nexus-data/javaprefs"
```

This configures **Java heap memory** and JVM parameters for performance:
- Sets initial and maximum heap sizes.
- Defines the direct memory size limit.
- Redirects Java preference storage to the persistent volume.

---

### Health Probes

Health checks to ensure the container is ready and healthy.
#### Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /service/rest/v1/status
    port: 8081
  initialDelaySeconds: 90
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
```
- Confirms Nexus is ready to receive traffic.
- Waits 90 sec. before the first check.
- Polls every 15s to detect readiness quickly.

#### Liveness Probe

```yaml
livenessProbe:
  httpGet:
    path: /service/rest/v1/status
    port: 8081
  initialDelaySeconds: 180
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
```
- Verifies the Nexus process remains healthy.
- Restarts the pod if health checks fail repeatedly.
- Uses a longer delay to accommodate Nexus startup time.

---

### Resource Management

Defines CPU and memory allocations.

```yaml
resources:
  requests:
    cpu: 1
    memory: 2Gi
  limits:
    cpu: 4
    memory: 4Gi
```
- `Requests`: Minimum guaranteed CPU and memory for stable performance.
- `Limits`: Prevents excessive resource consumption.

---

### Persistent Storage

Defines how persistent data is stored and mounted.

```yaml
volumeMounts:
  - name: nexus-data
    mountPath: /nexus-data
    readOnly: false
volumes:
  - name: nexus-data
    persistentVolumeClaim:
      claimName: nexus-pvc
      readOnly: false
```
- `volumeMounts`: Mounts the persistent storage inside the container at `/nexus-data`.
- `persistentVolumeClaim`: References `nexus-pvc`, ensuring repository data and configurations persist.

> **Note**: Without a PVC, all data would be lost when the pod restarts or rescheduled. This setup guarantees data durability across upgrades or Pod failures.

---

### Deploy and Verify

**Apply the StatefulSet manifest**:
```bash
kubectl apply -f statefulset.yaml
```

**Check the deployment status**:
```bash
kubectl -n nexus get statefulset
```

**You should see**:
```
NAME                 READY   AGE
nexus-statefulset    1/1     2m
```

Once deployed, Nexus will always attach to the same persistent volume after restarts or reschedules, ensuring data persists reliably.

Once running, Nexus will be accessible via the Service and maintain consistent storage across restarts.

---

## Expose Nexus Services

A Kubernetes **Service** provides a **stable network endpoint** for Pods. Since Pod IPs change during restarts, Services act as a consistent entry point, just like a restaurant phone number that stays the same even if the staff changes.

I’ll create two types of Services:

1. A **Service** to expose Nexus externally via IngressRoute.
2. A **headless Service** to support Pod discovery within the StatefulSet.

**File**: [`services.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/services.yaml)

**What it does**:

- Exposes the Nexus UI on port `8081` and Docker registry on port `5000`.
- Includes Prometheus annotations for later monitoring.
- Creates a **headless service** for internal DNS discovery by the StatefulSet.

**Apply the manifest**:
```bash  
kubectl apply -f services.yaml  
```

---

## Secure Nexus with HTTPS

In a previous project [**Secure K3s with Traefik and Cert-Manager**](https://www.hansterhorst.com/devops/secure-k3s-https/), I configured Treafik as an Ingress controller and installed Cert-manager with Helm for secure access to the Nexus UI with HTTPS.

Here, I’ll reuse the **production** Let’s Encrypt ClusterIssuer for Nexus.

### TLS Certificate

A **Certificate** tells the cert-manager which domain names to secure.

**File**: [`nexus-certificate.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/nexus-certificate.yaml)

**What it does**:

- Requests a Let’s Encrypt certificate via the `letsencrypt-production` ClusterIssuer.
- Creates a secret named `nexus-certificate` in the `nexus` namespace.
- Covers both domains:
    - `nexus.example.dev` for the Nexus UI.
    - `docker.nexus.example.dev` for the Docker registry.

---

### IngressRoute Configuration

The **IngressRoute** routes HTTPS traffic to the Nexus Service.

**Files**: [`nexus-ingressroute.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/nexus-ingressroute.yaml), [`docker-nexus-ingressroute.yaml`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/docker-nexus-ingressroute.yaml)

**What they do**:

- Define Traefik IngressRoutes for secure HTTPS traffic.
- Route requests from `nexus.example.dev` and `docker.nexus.example.dev` to the appropriate services.
- Attach the Let’s Encrypt TLS certificate managed by Cert-Manager.

**Apply the manifests**:
```bash  
envsubst < nexus-certificate.yaml | kubectl apply -f -  
envsubst < nexus-ingressroute.yaml | kubectl apply -f -  
envsubst < docker-nexus-ingressroute.yaml | kubectl apply -f -  
```

After applying the manifests, wait a few minutes to visit `https://nexus.example.dev`. You should see the Nexus UI secured with a valid Let’s Encrypt certificate.

---

## Deploy with Bash Script

To simplify deployment, I created an automated **Bash script** that applies all the necessary manifests in the correct order.

**File**: [`deploy-nexus.sh`](https://gitlab.com/devops8614042/nexus/-/blob/main/manifests/deploy-nexus.sh)

**What it does**:

1. Loads environment variables from `.env`.
2. Applies the Namespace and Storage resources.
3. Deploys the StatefulSet and Services.
4. Configures HTTPS via Cert-Manager and Traefik.

**Run the script**:
```bash  
bash deploy-nexus.sh  
```

This one-liner automates the complete Nexus setup, making redeployment or environment replication seamless.

---

## First Login

When Nexus starts for the first time, it generates a one-time admin password stored inside the Pod at `/nexus-data/admin.password`.

**List all Pods with there name**:
```bash  
kubectl -n nexus get pods -w  
```

**Get the admin password**:
```bash  
kubectl -n nexus exec -it <POD_NAME> -- cat /nexus-data/admin.password  
```

Log in at `https://nexus.example.dev` using:
- **Username**: `admin`.
- **Password**: Copy/paste the password from the console.

After logging in, change the default password and check the checkbox **disable anonymous access** for better security.

---

## What I Learned

Setting up **Nexus Repository Manager** on a **K3s cluster** taught me far more than just how to deploy an application. It gave me better insight into managing **stateful workloads**, **Kubernetes storage**, and **secure service exposure** in a production-like environment.

### Key Takeaways

- **Stateful Applications Require Careful Planning**  
  Learned how **StatefulSets**, **StorageClasses**, **PersistentVolumes (PV)**, and **PersistentVolumeClaims (PVC)** work together to maintain data persistence, even when Pods are rescheduled.
- **Local Storage Management Matters**  
  Using a **local directory** on the VPS (`/mnt`) gives me full control over where data is stored, simplifying maintenance and backups.
- **Kubernetes Security Contexts Are Crucial**  
  Running Nexus as a **non-root user** (`UID:GID 200`) reinforced container security best practices using the `securityContext` configuration.
- **Environment Variables Improve Reusability**  
  Managing configuration values in a `.env` file and using `envsubst` made the manifests **clean**, **portable**, and **reusable**across environments.
- **Certificates and Ingress Are Essential for Production**  
  Integrating **Traefik** and **Cert-Manager** provided automated TLS certificates and secure HTTPS access, which is critical for production-grade deployments.
- **Automation Simplifies Deployments**  
  Writing a **Bash script** to apply manifests in sequence reduced human error, ensured consistency, and reinforced the value of **Infrastructure as Code (IaC)**.

This project turned Kubernetes theory into a **practical, hands-on experience** and gave me a better understanding of how **persistence**, **security**, and **automation** come together in a cloud-native environment.

---

## Conclusion

I now have **Sonatype Nexus Repository** running on a **K3s cluster**, deployed as a **StatefulSet** with **persistent storage**, **Traefik Ingress**, and **TLS security** managed by **Cert-Manager**.

This setup provides a reliable, production-ready artifact repository. Ready to manage **Maven artifacts**, **npm packages**, and **Docker images** as part of my DevOps HomeLab.