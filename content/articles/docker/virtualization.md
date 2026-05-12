---
title: "Virtualization"
date: 2026-01-30
draft: false
tags: ["Containers", "Docker"]
weight: 2
---

# The Evolution of Virtualization

To understand Containers and how we got here, we need to understand how we moved from traditional physical server deployment to where we are now.

## Physical Servers

A physical server runs an operating system, such as Linux, and hosts multiple applications and binaries directly on the OS. If the server reaches capacity or an application consumes additional resources, you need more servers to run the other applications. This resulted in an entire data center.

```
┌────────────────────────────────┐  
│         Application 1          │
├────────────────────────────────┤
│         Application 2          │
├────────────────────────────────┤
│     Binaries and Libraries     │
├────────────────────────────────┤
│       Operating System         │
├────────────────────────────────┤
│          Hardware              │
└────────────────────────────────┘
```

This creates major problems:
1. **Dependency conflicts:** Applications often need different dependencies or operating systems
2. **Resource management:** Difficult to allocate CPU and RAM across competing applications on shared hardware
3. **Poor utilization:** Massive servers running a few applications waste resources
4. **System vulnerability:** A single application crash can bring down the entire server, and security attacks could access all applications
5. **Slow provisioning:** Ordering and setting up servers can take weeks.
6. **High capital costs:** Each new application requires purchasing dedicated hardware, increasing data center space, power consumption, and maintenance costs.

---

## Virtual Machines

With the problems of physical servers, the industry developed virtualization, which runs multiple small servers (virtual machines) on a single physical server.

```
┌─────────────────────────────────┐
│ ┌─────────────┐ ┌─────────────┐ │
│ │    App 1    │ │    App 2    │ │
│ ├─────────────┤ ├─────────────┤ │
│ │  Bins/Libs  │ │  Bins/Libs  │ │
│ ├─────────────┤ ├─────────────┤ │
│ │     OS      │ │     OS      │ │
│ └─────────────┘ └─────────────┘ │
├─────────────────────────────────┤
│           Hypervisor            │
├─────────────────────────────────┤
│            Host OS              │
├─────────────────────────────────┤
│            Hardware             │
└─────────────────────────────────┘
```

Hypervisors enabled this by sitting directly on the hardware and creating a virtualization layer that:

- Divides physical resources (CPU, RAM, storage, networking) into isolated virtual machines (VM)
- Each VM gets its own complete operating system
- Multiple VMs share the same physical server but remain completely isolated

Each VM has its own OS, which gives limitations:

- **Heavy**: Each VM needs a full OS, even for a small application
- **Slow boot**: Minutes to start because of booting an entire OS
- **Resource overhead**: Running 5 VMs means five complete operating systems consuming CPU and RAM
- **Not portable enough**: VMs are tied to specific hypervisors

---

## Containers

Given the limitations of VMs, the industry embraced containers after Docker simplified Linux Containers (LXC).

Instead of virtualizing hardware, containers virtualize the OS by sharing the same host kernel.

```
┌─────────────────────────────────┐
│ ┌─────────────┐ ┌─────────────┐ │
│ │ Container 1 │ │ Container 2 │ │
│ ├─────────────┤ ├─────────────┤ │
│ │  Bins/Libs  │ │  Bins/Libs  │ │
│ └─────────────┘ └─────────────┘ │
├─────────────────────────────────┤
│        Container Runtime        │
├─────────────────────────────────┤
│         Operating System        │
├─────────────────────────────────┤
│            Hardware             │
└─────────────────────────────────┘
```

This makes them:

- **Lightweight:** Only a few megabytes, only the packages and application, not the entire OS
- **Fast startup:** Seconds instead of minutes
- **Higher density:** More containers than VMs on the same server, to reduce overhead
- **Development-friendly:** Light enough to run multiple containers on a laptop
- **Efficient:** Shared kernel eliminates overhead

By sharing the host kernel, containers are less isolated. Linux containers require a Linux kernel. When running containers on macOS or Windows, you need a Linux virtual machine to host them.

With **Docker Desktop**, you create a Linux virtual machine and run containers inside it, providing the necessary container runtime.

---

## Physical Server vs. VMs vs. Containers

| Aspect       | Physical Server | Virtual Machine | Containers |
|--------------|-----------------|-----------------|------------|
| Isolation    | None            | Strong          | Good       |
| Startup      | Minutes         | Minutes         | Seconds    |
| Size         | N/A             | Gigabytes       | Megabytes  |
| Provisioning | Days/Weeks      | Minutes         | Seconds    |
| Utilization  | Low             | Good            | Excellent  |
| Development  | Not             | Heavy           | Ideal      |

---

## Conclusion

The evolution of virtualization from physical servers to VMs to containers reflects the industry's drive for efficiency and speed. Each step reduced overhead, improved portability, while trading some isolation.

- Physical servers wasted resources but offered complete isolation.
- Virtual machines improved utilization but added OS overhead.
- Containers maximized efficiency by sharing the host kernel.

Containers became the standard for deployment. They solve the classic **"works on my machine"** problem and enable fast deployment. Understanding how this technology works under the hood is essential for working with orchestration platforms like Kubernetes that manage containers in production.

---

## Definitions

**Physical Server**: A dedicated hardware server running a single operating system that hosts applications directly. Traditional deployment method before virtualization.

**Virtualization**: Technology that creates virtual versions of computing resources, allowing multiple isolated VMs to run on a single physical server.

**Virtual Machine (VM)**: A complete computer system running as software, with its own operating system, isolated from other VMs on the same physical server.

**Hypervisor**: Software that creates and manages virtual machines by dividing physical hardware resources (CPU, RAM, storage) among multiple VMs. Also called a Virtual Machine Monitor (VMM).

**Type 1 Hypervisor (Bare-metal)**: Hypervisor that runs directly on hardware without a host OS (VMware ESXi, Hyper-V, KVM).

**Type 2 Hypervisor (Hosted)**: Hypervisor that runs on top of an existing operating system (VirtualBox, VMware Workstation).

**Host OS**: The operating system running on the physical server that manages the hypervisor or container runtime.

**Guest OS**: The operating system running inside a virtual machine, isolated from the host OS.

**Container**: A lightweight, isolated process that shares the host OS kernel while maintaining its own filesystem and dependencies. Created from a container image.

**Container Runtime**: Software that runs containers (Docker, Podman, containerd). Manages container lifecycle and interfaces with the OS kernel.

**Isolation**: The separation between applications or systems to prevent interference. Physical servers have none, VMs have strong isolation, containers have good isolation.

**Resource Utilization**: How efficiently hardware resources (CPU, RAM, storage) are used. Containers achieve excellent utilization by eliminating OS overhead.

---