---
title: "Containers"
date: 2026-01-30
draft: false
tags: ["Containers", "Docker"]
weight: 3
---

# Containers

Containers have revolutionized the way we develop and deploy software. They provide a robust, efficient, and streamlined way to manage applications, dependencies, and configurations into a lightweight software package that runs consistently across any system.


> See my other article about [Virtualization](https://www.hansterhorst.com/articles/virtualization.md)

# Why Containers Exist

Containers solve problems in both the **"development of software"** and the **"deployment of the software"**: the classic **"it works on my computer!"** problem, and the many conflicts across different environments, dependencies, configurations, and OS versions during deployment.

## The Development Problem

When a new developer starts on the first day, they receive a laptop and spend a lot of time setting it up to run the company's software.

Working on multiple projects requires specific configurations. The setup process is time-consuming and causes constant issues with:

- Runtime versions (Java, Python, .NET).
- Database configurations.
- Credentials and dependencies.
- Version conflicts between projects.

### The Container Development Solution

With containers, these problems are solved by providing a complete package of the required software, including all dependencies, settings, credentials, and configurations.

Instead of installing everything on a local computer, containers package all the components needed to run a specific project into a single container image, so a developer only needs a **container runtime**, such as Docker, on their laptop.

With this runtime, the developer pulls the deployment image, runs it immediately, and works on the project in minutes.


---

## The Deployment Problem

Before containers, deploying software to a server was hard and time-consuming. Teams would set up the server with a different OS version, configure the right runtime environment, and install all the required packages.

When different applications required different versions of the programming language, the server needed both versions. Managing these requirements could cause problems.

### The Container Deployment Solution

Containers solve this problem by isolating applications from the underlying OS. They pack all required packages, with the correct versions and configurations, into a single package. This makes them portable and reduces environment-related problems.


## Containerized Environment

With a containerized environment, containers solve both sides of software deployment by providing a complete package that includes all required components.

The developer becomes productive within minutes, and the deployment of applications runs reliably across different environments.


---

## What is a Container?

A **Container** is a running instance of a **Container Image**, a lightweight, isolated process that runs the packaged application with all the required dependencies and configuration files consistently across different environments, such as a developer laptop, testing stage, or a production server.

The **Container Image** is a lightweight, standalone, executable software package that includes everything needed to run an application: the code, runtime, system tools, system libraries, and settings.

**Containers vs Images**:

- **Images**: Template that contains all required specifications
- **Containers**: Running instances of Images
- Both are lightweight portable packages

Unlike a virtual machine, which virtualizes hardware, a container virtualizes the operating system, sharing the same host OS kernel while maintaining isolated processes. This makes them faster to start.


---

## How Containers Work

Containers are isolated processes running on a host kernel. They use the Linux kernel to create isolated environments. This isolation happens through two main Linux kernel features:

```
┌─────────────────────────────────────┐
│ ┌─────────────────────────────────┐ │
│ │ ┌─────────────┐ ┌─────────────┐ │ │
│ │ │ Container 1 │ │ Container 2 │ │ │
│ │ └─────────────┘ └─────────────┘ │ │
│ └─────────────────────────────────┘ │
│ ┌─────────────────────────────────┐ │
│ │        Container Runtime        │ │
│ └─────────────────────────────────┘ │
│ ┌─────────────────────────────────┐ │
│ │           Linux Kernel          │ │
│ │ ┌─────────────┐ ┌─────────────┐ │ │
│ │ │  Namespaces │ │    Cgroups  │ │ │
│ │ └─────────────┘ └─────────────┘ │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```
### Namespaces

Namespaces are a feature of the Linux kernel that **isolates** processes by giving each namespace its own instance of system resources.

When you start a container, the kernel creates several isolated namespaces:

- **PID namespace**: Ensures that containers have their own process ID tree.
- **Network namespace**: Provides a container with its own virtual network interface, IP address, and port mappings.
- **Mount namespace**: **Isolates** the file system so the container sees only its own root directory.
- **UTS namespace**: **Gives** the container its own hostname, isolated from the host.
- **User namespace**: Maps container users to different host users.

Without namespaces, all processes on a Linux system share the same view of process IDs, network interfaces, filesystems, and hostnames.

#### Demonstration

The `pstree -p` command shows all running processes as a tree, starting with PID 1:

```
systemd(1)─┬─ModemManager(872)─┬─{ModemManager}(879)
           │                   ├─{ModemManager}(880)
           │                   └─{ModemManager}(882)
           ├─NetworkManager(850)─┬─{NetworkManager}(883)
           │                     ├─{NetworkManager}(884)
           │                     └─{NetworkManager}(885)
           ├─agetty(1061)
           ├─ ...
           ├─sshd(1010)───sshd-session(59098)───sshd-session(5912+
           ├─systemd(1208)─┬─(sd-pam)(1210)
           │               ├─dbus-daemon(1235)
           │               └─mpris-proxy(1234)
           ├─systemd-journal(334)
           ├─systemd-logind(795)
           ├─systemd-timesyn(406)───{systemd-timesyn}(423)
           ├─systemd-udevd(414)
           ├─ ...
           ├─winbindd(1110)─┬─wb-idmap(1138)
           │                └─wb[PI-INFRA](1116)
           └─wpa_supplicant(851)
```

When creating a new namespace, it will get a PID 1. This namespace doesn't know about other processes on the system and only shows its own.

**Create a new namespace**:
```bash
sudo unshare --fork --pid --mount-proc bash
```
This creates a new process and mounts the new namespace bash starting as PID 1.

```
root@pi-infra:/home/user#
```

**When running `pstree -p`**:
```
bash(1)───pstree(2)
```
Now it shows only:
- `bash`: PID 1
- `pstree`: Child process

This is a complete Linux system, but it doesn't know about the other host processes. It's a self-contained Linux environment that shares the same kernel, but it thinks it's the only process with PID 1.

> **Namespaces are automatically deleted when the last process exits.**


### Control Groups (cgroups)

While namespaces provide isolation, they don't limit resources. The bash namespace gets all the resources available on the Linux system.

Control groups (cgroups) are another Linux kernel feature that limits the resources a container can use:

- **CPU** percentage (e.g., 50% of one core)
- **Memory** maximum (e.g., 512 MB)
- **Disk I/O** (e.g., 100 MB/s read limit)

Without cgroups:

- One container could consume 100% of the CPU, **starving** other containers.
- A memory leak in one container **crashes** the entire host.
- A disk-intensive container **slows down** the entire system.

#### Demonstration

Cgroups organize processes into hierarchical groups and apply resource limits to those groups.

```
/sys/fs/cgroup/
├── bash
│    ├── cgroup.controllers
│    ├── ...
│    └── pids.peak
├── ...
└── user.slice
     ├── cgroup.controllers
     ├── ...
     └── user-1000.slice
          ├── cgroup.controllers
          ├── ...
          └── user@1000.service
```

Inside the bash namespace, create a cgroup.

**Create a cgroup in the specific directory**:
```bash
sudo mkdir /sys/fs/cgroup/devops
```

**Set a 10 MB memory limit**:
```bash
echo 10000000 | sudo tee /sys/fs/cgroup/devops/memory.max
```

> **Note**: On cgroups v2 systems, you may need to enable the memory controller first: 

 ```bash
 echo "+memory" | sudo tee /sys/fs/cgroup/devops.subtree_control
 ```

**Add the current shell to the cgroup**:
```bash
echo $$ | sudo tee /sys/fs/cgroup/devops/cgroup.procs
```
- `$$`: Is the process ID (PID) of the current shell.

> ***Tip***: Refer to the Bash manual `man bash` for more information.


**Test the memory limit**:
```bash
head -c 15M /dev/zero | tail
```
- `/dev/zero`: Is a special device file that produces an endless stream of null bytes (0x00)
- `head -c 15M`: Reads exactly 15 megabytes from this stream

This command allocates 15 MB of memory, but the system has a 10 MB memory limit. The Linux kernel kills the process with the error message **"Killed"**.

To delete the created **cgroup**, first exit the shell to start a new session.

**Remove the cgroup**:
```bash
sudo rmdir /sys/fs/cgroup/devops
```

Container runtimes (Docker, Podman) create a Linux namespace and apply cgroups. When a container exceeds its limits, the Linux kernel will automatically kill it.


---

### Union File System (OverlayFS)

A union filesystem is a filesystem that combines multiple directories (layers) stacked on top of each other as a single unified directory.

Container images are built as a stack of read-only layers, and, with a union filesystem, they are combined into a single, unified view.

#### How Union Filesystems Work

Imagine you have three separate directories with files:

- Directory A has file1.txt and **file2.txt**.
- Directory B has **file2.txt** and **_file3.txt_**.
- Directory C has **_file3.txt_** and file4.txt.

Without a union filesystem, you need to access each directory separately to get the files.

With a union filesystem, you layer all directories on top of each other. The layered view shows:

- file1.txt from A,
- file2.txt from B, hiding A's file2.txt
- file3.txt from C, hiding B's file3.txt
- file4.txt from C

When files have the same name, the upper layers hide the lower layers.

When you read a file from the layered view, the union filesystem searches from top to bottom, checking the upper layer (C) first, then the middle layer (B), then the lower layer (A), and returns the first match.

```
┌──────────────────────────────────────────┐
│      Union File System (OverlayFS)       │
│┌────────────────────────────────────────┐│
││                    file3.txt file4.txt ││ C
│├────────────────────────────────────────┤│
││          file2.txt file3.txt           ││ B
│├────────────────────────────────────────┤│
││ fil1.txt file2.txt                     ││ A
│└────────────────────────────────────────┘│
└──────────────────────────────────────────┘
```

The files aren't duplicated, the upper layers overriding the lower ones.

#### Container Image Structure

A container image is **a collection of read-only layers** stored as compressed archives. For example, an application image might have a:

- Layer-1 containing the Ubuntu base OS
- Layer-2 containing the Java/Python runtime
- Layer-3 containing application code.

Each layer is an archive containing files and directories.

#### OverlayFS in Running Containers

When you start a container, the container runtime stacks all the read-only layers and adds a new **writable layer** on top called the container layer. The container sees an OverlayFS combining all layers.

**Copy-on-Write (COW):** When you modify a file from a read-only layer, OverlayFS copies it to the writable image layer and modifies the copy there. The original file in the image layer remains unchanged.

**When the container stops:** The writable container layer is deleted. Image layers remain on disk and can be reused by other containers. This is why containers are ephemeral: changes persist only until the container is destroyed.

```
┌─────────────────────────────────┐
│        Container Image          │
│┌───────────────────────────────┐│
││ Container Layer (Read-Write)  ││← Changes written
│├───────────────────────────────┤│  here 
││ Application Layer (Read-Only) ││← Application
│├───────────────────────────────┤│  code
││ Runtime Layer (Read-Only)     ││← Java/Python
│├───────────────────────────────┤│
││ Base OS Layer (Read-Only)     ││← Ubuntu/Alpine
│└───────────────────────────────┘│
└─────────────────────────────────┘
```

**Benefits of OverlayFS:**

- **Storage efficiency:** Multiple containers share the same read-only image layers
- **Fast startup:** No copying needed, mount existing layers, and create a writable layer
- **Layer reuse:** Different images can share common base layers, reducing storage requirements
- **Immutability:** Image layers never change, ensuring consistency across deployments


OverlayFS is the third Linux kernel feature of container technology. Combined with namespaces (isolation) and cgroups (resource limits), containers become lightweight, fast, and efficient.

These three Linux kernel features are the foundation of container isolation. Docker, Podman, and Kubernetes are built on top of these features.


---

## Docker

Docker is a container platform that makes containers easy to use. It wraps the three Linux kernel features, **namespaces, cgroups, and OverlayFS**, into a user-friendly toolset:

- **Build**: Create container images from a Dockerfile
- **Ship**: Push and pull images to and from container registries (Docker Hub)
- **Run**: Start containers from container images

Before Docker, Linux Containers (LXC) existed, but they were:

- **Complex to use:** Manual namespace and cgroup configuration
- **Not portable:** Tied to specific Linux systems
- **No standardized image format:** Each system handled images differently
- **No easy distribution method:** No centralized registry

Docker simplified the deployment of containers with simple commands:

```bash
docker build -t myapp:1.0  # Build image
docker push myapp:1.0      # Upload to registry
docker run myapp:1.0       # Run container
```

### The Problem with Containers on Other OS

Because containers use Linux kernel features, they can only run on Linux systems.

To run containers on macOS or Windows computers, you need a Linux virtual machine (VM) running in the background. These operating systems use completely different kernels from Linux.

[Docker Desktop](https://www.docker.com/products/docker-desktop/) is a GUI application for macOS and Windows that creates a Linux VM to run containers.

Docker Desktop was previously free and open source, but in 2021, it changed its licensing model. There are alternatives to Docker Desktop:

- [Podman Desktop](https://podman-desktop.io/): Open source and uses almost the same Docker commands.
- [Rancher Desktop](https://rancherdesktop.io/): Open-source container management to run containers, and includes Kubernetes.

Docker made Linux containers accessible, simple, and popular, making them the standard for container deployment.


---
## Conclusion

Containers are Linux processes running inside namespaces with cgroup resource limits, using union filesystems for efficient layered storage.

Namespaces provide process **isolation** by creating separate views of system resources. All processes share the host kernel, there's no hardware virtualization.

Control groups are a Linux kernel resource management mechanism. While namespaces provide **isolation**, cgroups provide **resource guarantees**

Union filesystems are the third pillar of container technology, making containers efficient by enabling layer sharing. Combined with namespaces (isolation) and cgroups (resource limits), containers become lightweight, fast, and efficient.

Docker simplified containers by making them usable. Before Docker, containers were a complex Linux feature. After Docker, they became the standard deployment method.

----

## **Definitions**

**Container Image**: A read-only package containing application code, dependencies, and a minimal OS filesystem. The blueprint for containers.

**Container**: A lightweight, isolated process running with its own filesystem, created from a container image. Shares the host kernel.

**Docker**: A platform for building, running, and managing containers. Makes Linux kernel features accessible through simple commands.

**Docker Daemon**: The background service that manages containers, images, and networks.

**Hypervisor**: Software that creates and manages virtual machines (VMware, Hyper-V, KVM).

**Namespace**: Linux kernel feature providing process isolation. Containers use namespaces to appear isolated.

**Cgroup (Control Group)**: Linux kernel feature that limits and monitors resource usage for processes.

**Union File System**: A filesystem that layers multiple directories into one view. Enables efficient image storage and sharing.

**OCI (Open Container Initiative)**: Industry standard for container image format and runtime, ensuring images work across different tools.

**Docker Desktop**: Docker’s application for Mac/Windows/Linux with a GUI, CLI, and built-in Kubernetes.

**Docker Engine**: The core Docker daemon and CLI, available for Linux. Runs containers natively.

---
