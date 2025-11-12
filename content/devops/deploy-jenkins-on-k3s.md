---
title: Deploy Jenkins on K3s
date: 2025-11-12
tags: ["devops", "jenkins", "podman", "k3s"]
draft: false
weight: 7
---

After completing the **TWN DevOps Bootcamp**, my journey into DevOps didn't just end with theory and practice projects. It was the beginning of building my own **K3s DevOps Homelab** with the tools and concepts I'd learned. In my previous projects, I already:

1. Set up a **VPS Ubuntu server** as my base environment.
2. Installed **K3s**, a lightweight Kubernetes distribution, to manage containerized applications on the server.
3. Deployed **Nexus Repository Manager**, a repository manager to publish, share, and manage different artifacts.

In this project, I will deploy **Jenkins** on my cluster by following the official documentation from the Jenkins website. I'll explain the steps I took and highlight some additional implementations that worked better for me.

---

## What is Jenkins?

Jenkins is an automation server for Continuous Integration and Continuous Delivery (**CI/CD**). It automates the build, test, and deployment processes in software development. When a developer commits code changes to Git, Jenkins can automatically build the project and deploy it to various environments.

---

## Why Jenkins on a K3s Cluster?

In my daily work, I also use Jenkins for building artifacts during the development stage. By setting up my own DevOps environment, I can experiment with Jenkins and create CI/CD pipelines to build artifacts with the [Build Tools](https://gitlab.com/devops8614042/build-tools/-/tree/main?ref_type=heads) project and publish them on my **Nexus Repository Manager**.

It's also one of the most popular automation servers for managing CI/CD pipelines. Running Jenkins inside a K3s cluster makes it **scalable**, **resilient**, and aligned with **modern DevOps practices**.

---

## Replace Docker with Podman

During the **TWN DevOps Bootcamp**, I used Jenkins as a Docker container with Docker installed inside it (Docker-in-Docker). While this setup worked well for the course, it's not ideal in Kubernetes environments.

### Why Not Docker-in-Docker?

Since **Kubernetes v1.24**, the Docker runtime (`dockershim`) has been **deprecated and removed** from Kubernetes. Running Docker inside Kubernetes now requires mounting the Docker socket. This approach introduces security risks by granting root-level host access and adds unnecessary complexity.

### Custom Jenkins Image with Podman

After some research, I decided to replace Docker with **Podman** and built a **custom Jenkins container image** with **Podman pre-installed**. This setup allows Jenkins to build and manage containers directly within its own Pod, without mounting the Docker socket.

**How it works**:

1. Jenkins runs inside a single Pod on my K3s cluster using the custom image.
2. Pipeline jobs execute `podman` commands directly inside the Jenkins container.
3. Podman handles container builds and image management without requiring a background daemon.

**Benefits**:

- **Docker compatibility**: Podman uses the same command syntax as Docker.
- **Secure**: No Docker socket exposure.
- **Lightweight**: One self-contained Pod for Jenkins with Podman pre-installed.
- **Direct integration**: Jenkins runs builds in its own Pod without extra agents.
- **Daemonless**: Podman operates without a background daemon, reducing resource overhead and security risks.

This approach is perfect for my DevOps HomeLab and aligned with Kubernetes' move away from Docker.

---

## Prerequisites

If you want to follow along, make sure you have the following ready:

- A running **K3s** cluster.
- `kubectl` installed and configured locally.
- **Traefik** set up as the default Ingress controller.
- **Cert-Manager** installed, with a **ClusterIssuer** configured for automatic TLS certificate issuance.
- A registered **domain name** (e.g., `example.dev`).
- A DNS **CNAME record** for `jenkins` pointing to the domain.
- Port `8080` open, if you are using the UFW firewall.

---

## Using Environment Variables in K3s

To keep Kubernetes manifests modular, reusable, and secure, this project uses **environment variables** to inject configuration values during deployment.

Sensitive data, such as credentials, hostnames, and local file paths, are stored in a local `.env` file. The manifests reference these variables using placeholders like `${K3S_LOCAL_STORAGE}`, `${NODE_HOSTNAME}`, or `${ROOT_DOMAIN}`.

Here's an example deployment command:

```bash
envsubst < MANIFEST_FILENAME.yaml | kubectl apply -f -
```

This approach keeps the manifests **portable**, **environment-agnostic**, and free from **hardcoded values**.

> Read more: [Environment Variables in Kubernetes Manifests](https://www.hansterhorst.com/devops/k3s-environment-variables/)

---

## Install Jenkins on K3s

There are multiple ways to deploy Jenkins on Kubernetes:

- Using `kubectl` and **YAML** files to configure certain aspects of the deployment.
- Using `helm` and **Helm Chart** to manage a whole deployment.
- Using the **Jenkins Operator** to manage operations for Jenkins on Kubernetes in the cluster.

In this project, I will deploy Jenkins manually by using `kubectl` and Kubernetes manifest files. This is a great way to understand the different Kubernetes resources.

---

### What is a Kubernetes Manifest File?

A **Kubernetes manifest file** is a YAML file that describes the desired state of a resource within a Kubernetes cluster. Think of it as a blueprint: you define what you want (for example, a Deployment with three replicas of the application), and Kubernetes ensures that the cluster matches that state.

In this Jenkins deployment, I use several Kubernetes manifest files that work together to build and manage the complete application environment:

- **Namespace**: Provides logical isolation for Jenkins and its resources, keeping them separate from other workloads in the cluster.
- **Persistent Storage**: Ensures Jenkins' data survives Pod restarts or rescheduling.
    - **StorageClass**: Defines the type and behavior of the underlying storage (e.g., local or dynamic provisioning).
    - **PersistentVolume (PV)**: Represents the physical storage resource in the cluster.
    - **PersistentVolumeClaim (PVC)**: Requests and binds storage from a matching PersistentVolume for Jenkins' data directory.
- **StatefulSet**: Manages the Jenkins Pod with stable storage, consistent network identity, and ordered startup or shutdown behavior.
- **Kubernetes Service**: Provides networking access to Jenkins.
    - **Service**: Exposes the Jenkins application internally or externally to the cluster network.
    - **Headless Service**: Used by the StatefulSet to provide stable DNS names for Pods, enabling consistent communication.
- **HTTPS Traffic**: Secures external access to Jenkins.
    - **Certificate**: Manages TLS encryption for secure communication over HTTPS.
    - **IngressRoute**: Routes external traffic into Jenkins, enforcing HTTPS and domain-based access control.
- **Kubernetes Secret**: Stores base64-encoded sensitive credentials.

Together, these manifests define the complete deployment, allowing Jenkins to run **securely, persistently, and reliably**inside the Kubernetes cluster.

---

### Kubernetes Namespace

A **Namespace** is used for isolating groups of resources within a cluster. It's best practice to create namespaces for DevOps tools like **Jenkins**.

> **_File_**: [`namespace.yaml`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/namespace.yaml)

**What it does**:

- Declares a Namespace resource named `jenkins`.
- Isolates Jenkins from other workloads in the cluster.
- Simplifies resource management, access control, and network policies.

**Apply the manifest**:

```bash
kubectl apply -f namespace.yaml
```

---

### Kubernetes Persistent Storage

When running stateful applications on Kubernetes, such as **Jenkins**, it's essential to have a reliable way to manage data that must persist beyond the ephemeral lifecycle of pods. To ensure data durability across restarts, Kubernetes provides three key resources for persistent storage: **StorageClass**, **PersistentVolume (PV)**, and **PersistentVolumeClaim (PVC)**.

For this setup, I'll reuse the persistent storage configuration from my Nexus deployment, adjusting it to meet Jenkins' specific storage requirements.

To define the persistent storage resources, I'll create a manifest file named `volumes.yaml` and define the **PVC** and **PV**resources:

> **_File_**: [`volumes.yaml`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/volumes.yaml)

For more information, see the [Deploy Nexus Repository](https://www.hansterhorst.com/devops/deploy-nexus-on-k3s/#configure-persistent-storage) article on my website.

#### StorageClass (SC)

I also reuse the same **StorageClass** previously defined for the Nexus deployment.

Since it already exists in the cluster, there's no need to reapply it. This StorageClass defines how and where persistent volumes are provisioned.

> **_File_**: [`local-storageclass.yaml`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/local-storageclass.yaml)

**What it does**:

- Creates a `local-storage` class with `no-provisioner`.
- Uses `WaitForFirstConsumer` mode so the volume binds only when a Pod actually requests it.

#### Prepare the Storage Directory

Before applying the storage manifests, I'll create the local directory on my VPS and set the correct permissions:

```bash
ssh <USER>@<SERVER_IP>
sudo mkdir -p <K3S_LOCAL_STORAGE>/jenkins
sudo chown 1000:1000 -R <K3S_LOCAL_STORAGE>/jenkins
```

- `1000:1000`: Corresponds to the **Jenkins container's UID and GID** in the StatefulSet.
- `-R`: Applies ownership recursively.

**Check the node hostname**:

```bash
kubectl get nodes
```

Copy the value under the `NAME` column and use it as `${NODE_HOSTNAME}` in the manifests.

**Verify storage status**:

```bash
kubectl -n jenkins get pv
kubectl -n jenkins get pvc
```

#### Apply the Manifest File

Instead of hardcoding configuration values such as paths, hostnames, or credentials directly into the manifests, define them in a local `.env` file.

During deployment, these variables will be **automatically injected** into the manifests using the [`envsubst`](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html) command.

**Load the `.env` variables into the shell**:

```bash
set -a
source .env
set +a
```

This exports environment variables into the current shell, making them available to scripts and subprocesses.

**Apply the manifest to the cluster using `envsubst`**:

```bash
envsubst < volumes.yaml | kubectl apply -f -
```

This command substitutes environment variables before applying the configuration to your cluster.

Kubernetes persistent storage decouples application data from pod lifecycles, ensuring data durability and consistency across restarts. By combining **StorageClass**, **PersistentVolume**, and **PersistentVolumeClaim**, I can define and consume persistent storage resources dynamically or locally.

---

## Building a Custom Jenkins Image with Podman

Before deploying Jenkins to the cluster, I first need to build a **custom Jenkins image** that includes **Podman pre-installed**. This allows Jenkins to build and push container images directly from within its own Pod, without relying on Docker.

### Why a Custom Image?

The official Jenkins image doesn't include Podman by default. By extending it with Podman support, I can:

- Run Jenkins and Podman in **one self-contained Pod**.
- Execute `podman` commands **directly within Jenkins pipelines**.
- Keep the **setup lightweight and portable** for a single-node K3s environment.
- Use **Docker-compatible commands** through Podman, using the same command syntax as Docker.

### Creating the Dockerfile

With a Dockerfile, I will create a **custom Jenkins container image** with **Podman pre-installed**.

> **_File_**: [`Dockerfile`](https://gitlab.com/devops8614042/jenkins/-/blob/main/Dockerfile)

#### Starting with Jenkins 2.535

```dockerfile
FROM jenkins/jenkins:2.535-jdk21
LABEL description="Jenkins with Podman support"
```

- `FROM`: The build starts from the official **Jenkins 2.535 image** running on **JDK 21**, providing a fully functional Jenkins installation as the base layer.
- `LABEL`: For metadata and documentation within container registries.

#### Switching to Root for System Configuration

```dockerfile
USER root
```

Jenkins normally runs as a non-privileged `jenkins` user, but installing system packages requires root privileges. Switching to `root` allows installation and configuration steps that modify the container's base system.

#### Installing Podman and Dependencies

```dockerfile
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    podman \
    fuse-overlayfs \
    nftables \
    iptables && \
    rm -rf /var/lib/apt/lists/*
```

This updates the package repositories, upgrades system packages, and installs key packages:

- `--no-install-recommends`: Keeps the image smaller by skipping suggested packages.
- **`podman`**: Installs Podman (Docker replacement).
- **`fuse-overlayfs`**: Allows overlay filesystem in user space (needed for Podman storage).
- **`nftables`**: Modern Linux firewall/networking (used by Podman). **Netavark** is Podman's modern networking backend.
- **`iptables`**: Legacy firewall rules (compatibility/fallback).
- **`rm -rf /var/lib/apt/lists/*`**: Cleans up package cache to reduce image size.

#### Configuring Podman's Storage Backend

```dockerfile
RUN mkdir -p /etc/containers /var/lib/containers
```

These directories store Podman configuration and persistent data:

- `/etc/containers`: Configuration files.
- `/var/lib/containers`: Image and container storage.

The following `storage.conf` file defines how Podman manages container filesystems:

```dockerfile
RUN cat > /etc/containers/storage.conf <<'EOF'
[storage]
driver="overlay"
runroot="/var/run/containers/storage"
graphroot="/var/lib/containers/storage"

[storage.options.overlay]
mount_program="/usr/bin/fuse-overlayfs"
EOF
```

Creates storage configuration file:

- **`driver="overlay"`**: Uses overlay filesystem (efficient layering).
- **`runroot`**: Where Podman stores runtime state (temporary).
- **`graphroot`**: Where container images/layers are stored (persistent).
- **`mount_program`**: Uses fuse-overlayfs for mounting (works in containers).

#### Setting Container Runtime Behavior

```dockerfile
RUN cat > /etc/containers/containers.conf <<'EOF'
[containers]
netns="host"

[engine]
cgroup_manager="cgroupfs"
events_logger="file"

[network]
network_backend="netavark"
EOF
```

The `containers.conf` file defines how Podman's engine operates:

- **`[containers]` section**:
    - **`netns="host"`**: Built containers use the host's network namespace (simpler networking).
- **`[engine]` section**:
    - **`cgroup_manager="cgroupfs"`**: Uses cgroupfs for resource limits (systemd not available in containers).
    - **`events_logger="file"`**: Logs events to files instead of journald.
- **`[network]` section**:
    - **`network_backend="netavark"`**: Uses Podman's modern networking stack.

#### Running Jenkins as Root

```dockerfile
USER root
```
The `USER` directive keeps Jenkins running as root inside the container, which is required because **Podman cannot run in rootless mode within a Kubernetes Pod**.

Rootless Podman relies on Linux **user namespaces**, but Kubernetes already manages its own namespaces through the container runtime (like containerd or CRI-O). Since the Linux kernel doesn't support fully nested user namespaces, rootless Podman cannot function inside Kubernetes.

For my **K3s HomeLab**, this is acceptable because Jenkins runs in a **dedicated Kubernetes Namespace**, isolated from other workloads. As the cluster is self-managed and single-tenant, running Jenkins as root is a **safe and practical choice**.

### Building and Publishing the Image

Since I'm using Traefik with cert-manager for TLS certificates, my Nexus registry has proper HTTPS support, so I don't need to configure insecure registry settings.

**Build the image**:

```bash
podman build --arch amd64 -t docker.nexus.example.dev/jenkins-podman:2.535
```
- `--arch amd64`: The architecture of the VPS server is `x86_64`, which specifies the 64-bit Intel/AMD processor architecture.

**Log in to the Nexus registry**:

```bash
podman login docker.nexus.example.dev
```

**Push the image to Nexus**:

```bash
podman push docker.nexus.example.dev/jenkins-podman:2.535
```

Once pushed, the **custom Jenkins image with Podman** is available in my private Nexus repository and ready for deployment to the K3s cluster.

This setup fits modern Kubernetes environments, where Podman's **daemonless architecture** simplifies operations and avoids the security risks of Docker-in-Docker.

---

## Kubernetes Secret

A Kubernetes **Secret** is a special resource designed to securely store sensitive data within a cluster. Instead of embedding credentials directly into configuration files or container images, Secrets keep sensitive information separate, encrypted, and under stricter access control.

To pull images from the private Nexus repository, Kubernetes needs authentication credentials. I'll create a Secret of type `docker-registry` that stores these credentials securely.

### Creating the Secret

I will generate a manifest file with the `kubectl create secret` command. By using the `--dry-run=client` flag, the command executes without applying it to the cluster.

**Generate the Secret manifest using `kubectl`**:

```bash
kubectl create secret docker-registry docker-nexus-login \
  --docker-server=docker.nexus.example.dev \
  --docker-username=docker-nexus-username \
  --docker-password=docker-nexus-password \
  --docker-email=docker-nexus-email \
  -n jenkins \
  --dry-run=client -o yaml > manifests/docker-nexus-secret.yaml
```

**What this command does**:

- Creates a `docker-registry` type Secret named `docker-nexus-login`.
- Stores Nexus registry credentials (server, username, password, email).
- Places it in the `jenkins` namespace.
- Uses `--dry-run=client` to generate the YAML without applying it.
- Outputs the YAML to `manifests/docker-nexus-secret.yaml` file.

**Apply the Secret to your cluster**:

```bash
kubectl apply -f manifests/docker-nexus-secret.yaml
```

**Verify the Secret was created**:

```bash
kubectl get secrets -n jenkins
```

> **Note**: **Always** add Secret manifest files to `.gitignore` to keep them private.

With this Secret in place, Kubernetes can authenticate to the private Nexus registry and pull the custom Jenkins image during StatefulSet deployment. The Secret is referenced in the StatefulSet manifest using `imagePullSecrets`.

---

### Kubernetes StatefulSet

In the [official Jenkins documentation](https://www.jenkins.io/doc/book/installing/kubernetes/#kubernetes-jenkins-deployment), the example setup uses a **Deployment** manifest. However, Jenkins is a **stateful application** because it stores important data such as pipeline configurations, user accounts, and credentials.

While a **Deployment** is used for stateless applications, stateful applications like Jenkins require a **StatefulSet**. A StatefulSet provides stable pod identities and persistent storage, ensuring that even if a pod is rescheduled, it retains access to the same data and network identity.

A **StatefulSet** manages the lifecycle of stateful applications by providing:

- Stable, unique network identifiers for each pod.
- Consistent storage volumes that persist across restarts.
- Ordered deployment, scaling, and termination of pods.
- Reliable recovery after node restarts or pod failures.

To deploy Jenkins with Podman support, I'll use a StatefulSet manifest that runs as root to enable Podman functionality.

> **_File_**: [`statefulset.yaml`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/statefulset.yaml)

**What it does**:

- Deploys Jenkins as a single replica StatefulSet using the custom Jenkins image.
- Runs as root (`runAsUser: 0`) with privileged security context to support Podman.
- Mounts persistent storage at `/var/jenkins_home` for Jenkins' data and `/var/lib/containers/storage` for Podman images.
- Creates an in-memory volume at `/var/run/containers` for Podman runtime state.
- Defines readiness and liveness probes for health monitoring.
- Allocates CPU and memory resources for stability.

**Security considerations**:

While running as root is less secure than rootless alternatives, it's a pragmatic choice for a CI/CD environment because:

- It works reliably without complex user namespace configuration.
- Jenkins is isolated in its own namespace.
- The cluster is internal and not exposed to the internet.
- Many production CI/CD environments use similar configurations.

**Apply the manifest**:

```bash
envsubst < statefulset.yaml | kubectl apply -f -
```

**Check the status**:

```bash
kubectl -n jenkins get statefulset
kubectl -n jenkins get pods
```

Once deployed, Jenkins will have full Podman capabilities for building and pushing container images.

This **StatefulSet** runs Jenkins as a single Pod with persistent storage and Podman support, ensuring data and configuration persist through restarts. Reusing the same local-storage and PVC setup as Nexus keeps the deployment consistent, while health probes and resource limits make Jenkins stable and production-ready for CI/CD workloads in my K3s Homelab.

---

### Kubernetes Service

A Kubernetes **Service** provides a **stable network endpoint** for Pods. Since Pods are ephemeral and can be rescheduled with different IP addresses, Services ensure consistent access to a group of Pods performing the same task.

I'll create two types of Services:

1. A **Service** to expose Jenkins externally via IngressRoute.
2. A **headless Service** to support Pod discovery within the StatefulSet.

> **_File_**: [`services.yaml`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/services.yaml)

**What it does**:

- Exposes the Jenkins UI on port `8080`.
- Includes Prometheus annotations for future monitoring.
- Creates a **headless service** for internal DNS discovery by the StatefulSet.

**Apply the manifest to the cluster using `kubectl`**:

```bash
kubectl apply -f services.yaml
```

**Verify the Service is running**:

```bash
kubectl -n jenkins get services -o wide
```

This shows detailed information and uses port 8080 to access Jenkins in the browser.

```
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
jenkins-svc   ClusterIP   10.43.137.172   <none>        8080/TCP   20s   app=jenkins-server
```

Kubernetes **Services** are fundamental for reliable networking within a cluster. They abstract away Pod lifecycles, provide stable endpoints, and enable seamless communication both inside and outside the cluster. Whether you're building a microservices-based application or exposing a web application to the internet, understanding Services is essential to mastering Kubernetes.

---

## Secure Jenkins with HTTPS

In the [Secure K3s with Traefik and Cert-Manager](https://www.hansterhorst.com/devops/secure-k3s-https/#install-helm-for-cert-manager) project, I configured Treafik as an Ingress controller and installed Cert-manager with Helm for secure access to the Jenkins UI with HTTPS.

I will reuse the **production Let's Encrypt ClusterIssuer** for Jenkins.

### TLS Certificate

A **Certificate** tells cert-manager which domain names to secure.

> **_File_**: [`jenkins-certificate.yaml`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/jenkins-certificate.yaml)

**What it does**:

- Requests a Let's Encrypt certificate via the `letsencrypt-production` ClusterIssuer.
- Creates a secret named `jenkins-certificate` in the `jenkins` namespace.
- Covers the domain:
    - `jenkins.example.dev` (Jenkins UI).

---

### IngressRoute Configuration

The **IngressRoute** routes external HTTPS traffic to the Jenkins Service.

> **_File_**: [`jenkins-ingressroute.yaml`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/jenkins-ingressroute.yaml)

**What it does**:

- Defines Traefik IngressRoute for secure HTTPS traffic.
- Routes requests from `jenkins.example.dev`.
- Attaches the Let's Encrypt TLS certificate managed by Cert-Manager.

**Apply the manifests**:

```bash
envsubst < jenkins-certificate.yaml | kubectl apply -f -
envsubst < jenkins-ingressroute.yaml | kubectl apply -f -
```

After applying the manifests, wait a few minutes, then visit `https://jenkins.example.dev`. You should see the Jenkins UI secured with a valid Let's Encrypt certificate.

---

## Deploy with Bash Script

To simplify deployment, I created an automated **Bash script** that applies all the necessary manifests in the correct order.

> **_File_**: [`deploy-jenkins.sh`](https://gitlab.com/devops8614042/jenkins/-/blob/main/manifests/deploy-jenkins.sh)

**What it does**:

1. Loads environment variables from `.env`.
2. Applies the Namespace and Storage resources.
3. Deploys the StatefulSet and Services.
4. Configures HTTPS via Cert-Manager and Traefik.

**Run the script**:

```bash
bash manifests/deploy-jenkins.sh
```

This command automates the complete Jenkins setup, making redeployment or environment replication seamless.

---

## First-Time Jenkins Login

When Jenkins runs for the first time, it generates a random admin password. You can fetch it directly from the Pod.

**Check the Jenkins log file**:

```bash
kubectl -n jenkins get pods
kubectl -n jenkins logs <JENKINS_POD_NAME>
```

**Or interact directly with the Pod container**:

```bash
kubectl -n jenkins exec -it <JENKINS_POD_NAME> -- cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the password and open the browser to access the Jenkins dashboard.

**Access Jenkins in the browser**:

```
https://jenkins.example.dev
```

Once you enter the password, follow the on-screen steps to install the suggested plugins and create an admin user.

---
## Building Container Images with Podman

Now that Jenkins is deployed with Podman support, I can create CI/CD pipelines that build container images and push them to my private Nexus repository, completing the full continuous integration workflow.

### Why Podman Instead of Docker?

During the **TWN DevOps Bootcamp**, I created images with **Docker**, but now I use Podman because Docker is difficult to set up in a K3s cluster because it requires either mounting the Docker socket or running in privileged mode. The Docker-in-Docker approach introduces security risks by granting root-level host access and adds unnecessary complexity in Kubernetes environments.

### Credentials in Jenkins

In Jenkins, credentials play a key role in automation. Rather than storing passwords or API tokens directly in pipelines, Jenkins keeps them securely in its internal store and injects them into jobs only when needed. This keeps sensitive information out of source code and logs and simplifying credential management.

Every credential in Jenkins has a **scope**, which defines where and how it can be accessed:

- **System Scope**: Used internally by Jenkins for global integrations such as Git, Docker registries, or Kubernetes clusters. Jobs or pipelines cannot access credentials with this scope.
- **Global Scope**: Accessible across all jobs and pipelines in Jenkins. This is the most common scope for secrets used in pipelines or freestyle projects.

To create a new credential in Jenkins:

1. In the **Jenkins UI**, navigate to **Manage Jenkins** → **Credentials**.
2. Under **Store ↓**, click the **System** link.
3. Click **(Global credentials)** under the **Domains** column to open the **Global credentials**.
4. Click **+ Add Credentials** and configure the following options:
    - **Kind**: Select the kind of credential, `Username with password`.
    - **Scope**: Select **Global** or **System**.
    - **Username**: `username`.
    - **Treat username as secret**: Enable this to hide the username in logs.
    - **Password**: `password`.
    - **ID**: `id-credentials`.
5. Click **Create** to save the credential.

Once created, it appears under the **System** → **Global credentials**. You can then reference it securely in the pipeline:

```groovy
withCredentials([usernamePassword(credentialsId: 'id-credentials', 
    usernameVariable: 'USER', passwordVariable: 'PASS')]) {
    sh "echo $PASS | docker login -u $USER --password-stdin"
}
```

- `withCredentials(...)`: Securely injects stored credentials into the build environment.
- `echo $PASS | ... --password-stdin`: Pipes the password via standard input to prevent it from appearing in logs.

---

## Creating a Jenkins Pipeline

With **Podman** installed, **Maven tools configured in Jenkins**, and credentials properly set up, I can now create a Jenkins pipeline that automates the process of building a Maven application, generating container images, and pushing them to **Nexus**.

### Prerequisites

Before creating the pipeline, ensure the following is configured in Jenkins:

- **Maven Tool**: Configure Maven 3.9 in Jenkins under **Manage Jenkins** → **Global Tool Configuration** → **Maven installations**. Set the name as `maven-3.9`.
- **Credentials**: Create the necessary credentials (Nexus Docker registry and GitLab) as described in the previous section.
- **Dockerfile**: The project should include a Dockerfile that accepts `MAVEN_VERSION` as a build argument to copy the correct JAR file.

---

## Pipeline Breakdown

This pipeline is designed to work within a **Multibranch Pipeline** project in Jenkins. Each branch in the repository contains its own `Jenkinsfile`, allowing branch-specific logic, build parameters, and deployment rules.

In this case, the pipeline configuration shown below applies specifically to the **current branch**, meaning Jenkins automatically detects and runs this workflow only when changes are made to this branch.

This approach ensures isolated build environments, version consistency, and independent deployment cycles for each branch.

> **_File_**: [maven-podman-pipeline](https://gitlab.com/devops8614042/build-tools/-/blob/maven-podman-pipeline/Jenkinsfile)

### Base Configuration

```groovy
pipeline {
   agent any
	tools {
	   maven "maven-3.9"
	}
	parameters {
	   choice(
	           name: 'VERSION_INCREMENT',
	           choices: ['patch', 'minor', 'major'],
	           description: 'Version increment type (patch, minor, major)'
	   )
	   booleanParam(
	           name: 'KEEP_SNAPSHOT',
	           defaultValue: true,
	           description: 'Keep -SNAPSHOT suffix'
	   )
	}
	environment {
	   PROJECT_DIRECTORY = './react'
	   NEXUS_URL = credentials('nexus-url')
	}
   stages {
      // the stage steps to build and publish the application
   }
}
```

- Defines the base structure of the pipeline.
- Uses the default Jenkins agent (`any`) and configures Maven 3.9 as the build tool.
- Declares build parameters for version increment type and SNAPSHOT handling.
- Sets environment variables for the project directory and Nexus URL.
- Contains the main `stages` section where the build and deployment logic resides.

### Test Maven Application

```groovy
stage('Test Maven Application') {
   steps {
      dir("${PROJECT_DIRECTORY}") {
         script {
            echo 'Start testing the Maven application...'
            sh 'mvn test'

            def pom = readFile('pom.xml')
            def artifactId = (pom =~ '<artifactId>(.+?)</artifactId>')[1][1]
            def version = (pom =~ '<version>(.+?)</version>')[1][1]

            env.APP_NAME = artifactId
            env.APP_VERSION = version
            echo "App: ${APP_NAME}-${APP_VERSION}"
         }
      }
   }
}
```

- Runs Maven tests to verify the application works correctly.
- Reads the `pom.xml` file and extracts the artifact ID and version using regex patterns.
- Stores these values as environment variables (`APP_NAME` and `APP_VERSION`) for use in the following stages.

> **_Note_**: Uses `[1][1]` to get the **second match** because this POM has a parent section. The first match `[0]` would be the parent's values, while `[1]` captures the project's actual artifactId and version.

### Increment Maven Version

```groovy
stage('Increment Maven Version') {
  steps {
    dir("${PROJECT_DIRECTORY}") {
       script {
          echo 'Start increment application version...'
          echo "Increment type: ${params.VERSION_INCREMENT}"
          def SNAPSHOT = 'SNAPSHOT'
          def isSnapshot = APP_VERSION.contains(SNAPSHOT)
          def snapshotSuffix = (isSnapshot && params.KEEP_SNAPSHOT) ? \
            "-${SNAPSHOT}" : ''
          // Use switch statement to handle version increment types
          switch (params.VERSION_INCREMENT) {
             case 'major':
                sh """
                   mvn build-helper:parse-version versions:set \  
                   -DnewVersion=\\\
                    ${parsedVersion.nextMajorVersion}.0.0${snapshotSuffix} \  
                   versions:commit
                   """
                break
             case 'minor':
                sh """
                   mvn build-helper:parse-version versions:set \  
                   -DnewVersion=\\\${parsedVersion.majorVersion}.\\\  
                   ${parsedVersion.nextMinorVersion}.0${snapshotSuffix} \  
                   versions:commit
                   """
                break
             case 'patch':
             default:
                sh """
                   mvn build-helper:parse-version versions:set \
                   -DnewVersion=\\\${parsedVersion.majorVersion}.\\\
                   ${parsedVersion.minorVersion}.\\\
                   ${parsedVersion.nextIncrementalVersion}${snapshotSuffix}
                   versions:commit
                   """
                break
          }
          def pom = readFile('pom.xml')
          def version = (pom =~ '<version>(.+?)</version>')[1][1]
          env.APP_VERSION = version
          env.MAVEN_VERSION = "${APP_NAME}-${APP_VERSION}"
          echo "App: ${MAVEN_VERSION}"
       }
    }
  }
}
```
- Automatically bumps the version number in `pom.xml` based on the selected increment type (e.g., 1.2.3 → 1.2.4 for a patch).
- Detects if the current version is a SNAPSHOT and preserves that suffix if the parameter is enabled.
- Uses a switch statement to handle major, minor, or patch version increments.
- Leverages Maven's `build-helper:parse-version` and `versions:set` plugins to increment the version.
- Reads the updated `pom.xml` to get the new version number.
- Creates a `MAVEN_VERSION` variable combining the `APP_NAME` and `APP_VERSION` for the Podman build.

### Build Maven JAR File

```groovy
stage('Build Maven JAR file') {
   steps {
      dir("${PROJECT_DIRECTORY}") {
         script {
            echo 'Start building JAR file...'
            sh 'mvn clean install -DskipTests'
            sh 'ls target/*.jar'
         }
      }
   }
}
```

- Executes `mvn clean install` to compile the application and package it as a JAR file.
- Uses `-DskipTests` to skip test execution (already run in the test stage).
- Lists the generated JAR files in the `target/` directory for verification.

### Build Maven Image with Podman

```groovy
stage('Build Maven Image with Podman') {
   steps {
      dir("$PROJECT_DIRECTORY") {
         script {
            echo 'Start building Podman image...'
            env.IMAGE_VERSION = "${APP_NAME}:${APP_VERSION}"
            env.NEXUS_REPO = "${NEXUS_URL}/${IMAGE_VERSION}"
            echo "NEXUS_REPO: ${NEXUS_REPO}"
            sh """
              podman build --build-arg MAVEN_VERSION=${MAVEN_VERSION} \
              -t ${NEXUS_REPO} .
            """
         }
      }
   }
}
```
- Builds a container image using Podman and the Dockerfile in the project directory (`.`).
- Passes `MAVEN_VERSION` as a build argument for the Dockerfile to copy the correct JAR.
- Tags the image with the format: `docker.nexus.example.dev/maven:version`.


### Publish Podman Image to Nexus

```groovy
stage('Publish Podman Image to Nexus') {
   steps {
      script {
         echo 'Start pushing image to Nexus...'
         withCredentials([usernamePassword(
                 credentialsId: 'nexus-docker',
                 passwordVariable: 'PASS', usernameVariable: 'USER')]) {
            sh """
               echo "${PASS}" | podman login ${NEXUS_URL} -u ${USER} \
               --password-stdin
               podman push ${NEXUS_REPO}
            """
         }
      }
   }
}
```
- Authenticates to the Nexus Docker registry using credentials stored in Jenkins.
- Uses `--password-stdin` for secure password handling, preventing password exposure in logs.
- Pushes the tagged image to Nexus.
- Credentials are injected via `withCredentials` without appearing in console logs.

### Commit New POM Version to Git

```groovy
stage('Commit new pom version to Git') {
   steps {
      dir("$PROJECT_DIRECTORY") {
         script {
            echo 'Start committing new pom version...'

            def gitBranchUrl = GIT_URL.replaceFirst('^https://', '')

            withCredentials([usernamePassword(
                    credentialsId: 'gitlab-credentials',
                    passwordVariable: 'PASS', usernameVariable: 'USER')]) {
               sh 'git config --global user.email "jenkins@example.com"'
               sh 'git config --global user.name "jenkins"'
               sh "git remote set-url origin \
                    https://${USER}:${PASS}@${gitBranchUrl}"
               sh 'git add pom.xml'
               sh 'git commit -m "Increased pom version by Jenkins"'
               sh "git push origin HEAD:${GIT_BRANCH}"
            }
         }
      }
   }
}
```
- Commits the updated `pom.xml` with the incremented version back to Git.
- Configures Git identity for the Jenkins user.
- Strips `https://` from the Git URL and reconstructs it with embedded credentials for authentication.
- Adds, commits, and pushes the `pom.xml` changes to the current branch.

This pipeline provides a **complete end-to-end CI/CD flow**, from testing and version management to artifact packaging, containerization, and committing the changes back to Git.

### Start the Pipeline

Trigger a build manually:

1. Navigate to the pipeline job.
2. Click **Build Now**.
3. Click on the build number to view the console output.
4. Watch as Jenkins builds and pushes your container image.

**Verify in Nexus**:

After a successful build, verify the image was pushed:

1. Go to the **Nexus UI**.
2. Navigate to **Browse** → **docker-hosted**.
3. You should see the image with the build number tag.

With this pipeline in place, I now have a fully automated CI/CD workflow that builds, tests, versions, containerizes, and publishes my Maven applications.

---

## What I Learned

I had already worked with Kubernetes resources in my Nexus deployment, but this project took me several steps further by applying those concepts to a more complex setup with Jenkins and container image builds.

Building a **custom Jenkins image with Podman** was a real challenge. I tested different approaches and ran into many errors, especially with Jenkins agents and their interaction with other container runtimes. Debugging these issues taught me a lot about container runtime behavior, privileged Pods, and network namespaces.

Replacing Docker with Podman showed me the advantages of daemonless tools in Kubernetes environments. Despite the difficulties, seeing everything work together was incredibly rewarding.

Managing storage and Podman image data taught me how to handle cleanup and resource management effectively, while deploying Jenkins manually showed how all the pieces come together into a secure and reliable CI/CD environment.

---

## Conclusion

Deploying Jenkins with Podman on my **K3s DevOps HomeLab** gave me a secure, efficient, and fully automated **CI/CD environment**. This setup brings together key Kubernetes concepts, from storage and secrets to networking and ingress, into a production-ready workflow.

By replacing Docker with **Podman**, I aligned my setup with modern Kubernetes practices, improved security, and simplified container management. My DevOps HomeLab now serves as a foundation for exploring other DevOps topics such as **GitOps**, **monitoring**, and create full-stack projects and use Jenkins and Nexus to deploy them.

With Jenkins, I can build container images with Podman, push them to my private Nexus registry, and orchestrate the entire software delivery pipeline, all within my own Kubernetes cluster.

Most importantly, this project reminded me that the best way to learn DevOps is by building and experimenting with real systems. Every error, misconfiguration, and troubleshooting became part of the learning process, turning theory into hands-on practice.
