---
title: Security Context
date: 2026-03-14
weight: 12
tags: [Security Context, Security]
---
# Security Context

Security settings that control how a container runs can be configured in Kubernetes at the Pod level or the container level. If both are defined, the container-level settings override the  Pod-level settings.

The equivalent Docker flags are:

```bash
docker run --user=1001 ubuntu
docker run --cap-add=MAC_ADMIN ubuntu
docker run --cap-drop=KILL ubuntu
docker run --privileged ubuntu
```

> See also: [Docker Security](http://hansterhorst.com/notes/docker/security/)

---

## Pod-Level Security Context

To configure security settings at the  Pod level, add a `securityContext` field under `spec`. Settings defined here apply to all containers in the  Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1001
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
```

**Verify the user ID**
```bash
kubectl exec -it web-pod -- id
# uid=1001 gid=0(root) groups=0(root)
```

---

## Container-Level Security Context

To apply security settings to a specific container, move the `securityContext` block inside the container specification. Container-level settings override  Pod-level settings when both are defined.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1001
```

**Verify the user ID**
```bash
kubectl exec -it web-pod -c ubuntu -- id
# uid=1001 gid=0(root) groups=0(root)
```

---

## Linux Capabilities

Linux capabilities can only be configured at the container level, not at the  Pod level. Use the `capabilities` field under `securityContext` to `add` or `drop` capabilities:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1001
        capabilities:
          add: ["MAC_ADMIN", "NET_ADMIN"]
          drop: ["KILL"]
```

**Verify the capabilities**
```bash
kubectl exec -it web-pod -c ubuntu -- cat /proc/1/status | grep Cap
```
```
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 00000002a80435db
CapAmb: 0000000000000000
```

**Decode the hexadecimal value**
```bash
capsh --decode=00000002a80435db
```
```
0x00000002a80435db=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_admin,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap,cap_mac_admin
```

This output lists the capabilities active inside the container.

> **Note**: `/usr/include/linux/capability.h` is a Linux kernel header file that defines all capabilities available on a Linux system.

---

## Reference

| Setting                             | Level            | Description                                               |
|-------------------------------------|------------------|-----------------------------------------------------------|
| `securityContext.runAsUser`         | Pod or container | Sets the UID for the process running inside the container |
| `securityContext.capabilities.add`  | Container only   | Adds Linux capabilities                                   |
| `securityContext.capabilities.drop` | Container only   | Drops Linux capabilities                                  |
---
