---
title: Docker Security
date: 2025-03-14 
---

# Docker Security

Docker containers are not completely isolated from their host. Containers and the host share the same kernel, and containers are isolated using Linux namespaces.

```
┌─────────────────────────────────────────┐
│ ┌─────────────────────────────────────┐ │
│ │ ┌─────────────────────────────────┐ │ │
│ │ │                                 │ │ │
│ │ │       Container Namespace       │ │ │
│ │ └─────────────────────────────────┘ │ │
│ │                                     │ │
│ │           Host Namespace            │ │
│ └─────────────────────────────────────┘ │
│                                         │
│                Kernel                   │
└─────────────────────────────────────────┘
```

The host has a namespace, and the containers have their own namespaces. All processes run by containers are, in fact, run on the host itself, but within their own namespace.

---

## Process Isolation

A Docker container has its own namespace and sees only its processes. The output shows the `sleep 3600` process with PID one.

**Container processes**
```bash
docker run ubuntu sleep 3600
docker exec ubuntu ps aux
```

```
USER         PID %CPU %MEM   ... COMMAND
root           1  0.0  0.0   ... sleep 3600
root          39  0.0  0.0   ... ps aux
```

On the host, all processes, including container processes, are visible as separate system processes. The host shows the same `sleep 3600` command with a different PID.

**Host processes**
```bash
ps aux | grep sleep
```
```
USER     PID    %CPU %MEM  ... COMMAND
root         1  0.0  0.1   ... /sbin/init
...
pi       45528  0.0  0.3   ... docker run ubuntu sleep 3600
root     45577  0.0  0.0   ... sleep 3600
pi       46241  0.0  0.0   ... grep --color=auto sleep
```

Different namespaces can assign different PIDs, enabling Docker container isolation.

---

## Users and Security

The Docker host has a set of users: a root user and a number of non-root users. By default, Docker runs processes within containers as the root user.

If you do not want the process within the container to run as root, you have two options.

1. You can set the user using the `--user` option within the `docker run` command and specify a new user ID.
```bash
docker run --user=1001 ubuntu sleep 3600 
```

2. You can enforce this at the image level by using the `USER` instruction in the Dockerfile and a specific user ID.
```dockerfile
FROM ubuntu
USER 1001
```

Building and running that custom image will execute the process with the specified user ID without passing it at runtime.

```bash
docker exec -it user-ubuntu ps aux
```
```
USER         PID %CPU %MEM   ... COMMAND
1001           1  0.0  0.0   ... sleep 3600
1001          15 20.0  0.0   ... ps aux
```

---

## Linux Capabilities and the Root User

If containers run as the root user by default, is the root user inside the container the same as the root user on the host?

No, Docker applies security mechanisms that restrict what the root user inside the container can do, primarily through Linux capabilities.

The root user on the host Linux system has unrestricted access, including modifying files and permissions, access control, creating or killing processes, setting group or user IDs, etc., and system operations like rebooting the host.

By default, Docker runs a container with a limited set of these capabilities. Processes within the container do not have the privileges to reboot the host or perform operations that could disrupt the host or other containers.

---

## Modifying Capabilities

If you need to override the default behavior, Docker provides several options. You can grant additional privileges using the `--cap-add` option in the `docker run` command.

```bash
docker run --cap-add=MAC_ADMIN ubuntu
```

You can remove privileges using the `--cap-drop` option.

```bash
docker run --cap-drop=KILL ubuntu
```

And if you need to run a container with all privileges enabled, you can use the `--privileged` flag.

```bash
docker run --privileged ubuntu
```

> **_Note_**: The file `/usr/include/linux/capability.h` is a Linux kernel header file that defines all capabilities available on a Linux system.

---
