---
title: Falco Runtime Security
date: 2026-11-13
weight: 9
draft: false
reviewed: true
tags:
  - CKS
  - Security
  - Kubernetes
  - Falco
  - Runtime Security
  - Tutorial
---
# Project Falco Runtime Security

In this project, I installed Falco on every node in my homelab cluster, simulated an attack on a running container, and monitored the alerts. Before the change, the cluster had no checks inside containers to track activity. Opening a shell, reading `/etc/shadow`, or changing a web page left no trace. Afterward, Falco reports each action, along with the Pod, Namespace, and command.

The other CKS projects protect the cluster before a Pod starts by using RBAC, admission policies, and signed images. But they cannot detect an attacker who gains access to a running container. Falco monitors system calls made by every process on the node and checks them against its rules. I then make the container immutable, causing part of the same attack to fail.

---

## The Plan

Falco runs as a DaemonSet with a Pod on each node. It uses an eBPF probe in the kernel to read the system calls, the same technique that Cilium uses for networking. I run an nginx Pod on the worker node and attack it with `kubectl exec`.

The project uses these names and values:

| Item          | Value                                                          |
| ------------- | -------------------------------------------------------------- |
| Falco         | Helm chart `falcosecurity/falco` in the Namespace `falco`      |
| Driver        | `modern_ebpf`                                                  |
| Custom rule   | `Write to nginx web root`, in `falco-values.yaml`              |
| Target        | Pod `web` with the image `nginx`, on `k8s-worker-1`            |
| Namespace     | `runtime`                                                      |

---

## How Falco Works

Falco monitors every system call on the node, such as opening a file, starting a process, or using a network connection. It adds container and Pod details to each call, then checks it against its rules. A matching rule triggers an alert.

A rule has five main fields. This is a simplified version of a default rule:

```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint or exec point into a container with an attached terminal.
  condition: spawned_process and container and shell_procs and proc.tty != 0
  output: A shell was spawned in a container with an attached terminal (...)
  priority: NOTICE
```
- `rule`: The name of the rule.
- `desc`: A description of what the rule detects.
- `condition`: The filter on the system call. `spawned_process` and `container` are macros, reusable parts of a condition.
- `output`: The alert text. Fields such as `%proc.cmdline` and `%k8s.pod.name` add the details.
- `priority`: The severity, from `DEBUG` to `EMERGENCY`.

Falco includes a set of stable rules in `falco_rules.yaml`. The Helm chart loads and maintains these rules. Custom rules should be placed in a separate file, which is not overwritten when the stable rules are updated.

---

## Step 1: Install Falco

The Helm chart runs Falco as a DaemonSet. By default, it tolerates the `node-role.kubernetes.io/control-plane:NoSchedule` taint, and runs on the control plane nodes too.

**Add the Falco Helm repository**
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
```

**Update the repositories**
```bash
helm repo update
```

**Install Falco**
```bash
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf \
  --set tty=true
```
- `driver.kind=modern_ebpf`: Reads the system calls with the eBPF probe that is built into Falco. No kernel module, no compiler on the node.
- `tty=true`: Writes each alert to the log.

**Wait until the Falco Pods are ready**
```bash
kubectl -n falco get pods -o wide
```
```
NAME          READY   STATUS    RESTARTS   AGE   IP           NODE
falco-28vsv   2/2     Running   0          28s   10.0.0.184   k8s-worker-1
falco-9f24t   2/2     Running   0          28s   10.0.2.84    k8s-master-3
falco-cw899   2/2     Running   0          28s   10.0.3.250   k8s-master-1
falco-frqjv   2/2     Running   0          28s   10.0.1.15    k8s-master-2
```

The DaemonSet runs one Falco Pod on every node.

**Save the name of the Falco Pod on the worker node**
```bash
FALCO_POD=$(kubectl -n falco get pods --field-selector spec.nodeName=k8s-worker-1 -o name)
```

> **_Note:_** The Falco docs now recommend the Falco Operator for Kubernetes. The Helm chart stays fully supported.

---

## Step 2: Attack the Container

The attack needs a target, such as a plain nginx Pod running on the worker node. The Falco Pod on that node can observe every system call the attack makes.

**Create the Namespace**
```bash
kubectl create namespace runtime
```

**Run nginx on the worker node**
```bash
kubectl -n runtime run web --image=nginx \
  --overrides='{"spec":{"nodeName":"k8s-worker-1"}}'
```

The attack follows a common pattern: get a shell, read credentials, and run a tool that the image doesn't contain.

**Open a shell in the container**
```bash
kubectl -n runtime exec -it web -- bash
```

Run these commands in the shell:

**Read the password hashes**
```bash
cat /etc/shadow
```

**Copy a binary and run it**
```bash
cp /bin/ls /tmp/scan && /tmp/scan /
```

**Change the web page**
```bash
echo hacked > /usr/share/nginx/html/index.html
```

**Leave the shell**
```bash
exit
```

---

## Step 3: Read the Alerts

Falco writes the alerts to the log of its own Pod. Only the Pod on the node of the attack sees the system calls.

**Show the alerts of the Falco Pod on the worker node**
```bash
kubectl -n falco logs $FALCO_POD -c falco | grep k8s_ns_name=runtime
```
```
21:45:33.379489636: Notice A shell was spawned in a container with an attached terminal | evt_type=execve user=root user_uid=0 user_loginuid=-1 process=bash proc_exepath=/usr/bin/bash parent=containerd-shim command=bash terminal=34816 exe_flags=EXE_WRITABLE|EXE_LOWER_LAYER container_id=795c4790cf1d container_name=web container_image_repository=docker.io/library/nginx container_image_tag=latest k8s_pod_name=web k8s_ns_name=runtime
21:45:39.161753990: Warning Sensitive file opened for reading by non-trusted program | file=/etc/shadow gparent=containerd-shim ggparent=systemd gggparent=<NA> evt_type=openat user=root user_uid=0 user_loginuid=-1 process=cat proc_exepath=/usr/bin/cat parent=bash command=cat /etc/shadow terminal=34816 container_id=795c4790cf1d container_name=web container_image_repository=docker.io/library/nginx container_image_tag=latest k8s_pod_name=web k8s_ns_name=runtime
21:45:51.586774603: Critical Executing binary not part of base image | proc_exe=/tmp/scan proc_sname=bash gparent=containerd-shim proc_exe_ino_ctime=1795470351583206699 proc_exe_ino_mtime=1795470351583206699 proc_exe_ino_ctime_duration_proc_start=2605851 proc_cwd=/ container_start_ts=1795470317292864695 evt_type=execve user=root user_uid=0 user_loginuid=-1 process=scan proc_exepath=/tmp/scan parent=bash command=scan / terminal=34816 exe_flags=EXE_WRITABLE|EXE_UPPER_LAYER container_id=795c4790cf1d container_name=web container_image_repository=docker.io/library/nginx container_image_tag=latest k8s_pod_name=web k8s_ns_name=runtime
```

The output shows three alerts from the stable rules. The log shows the output text of a rule, not its name.

| Rule                                      | Alert text                                                   | Priority   | Action                       |
| ----------------------------------------- | ------------------------------------------------------------ | ---------- | ---------------------------- |
| Terminal shell in container               | A shell was spawned in a container with an attached terminal | `NOTICE`   | `kubectl exec -it ... bash`  |
| Read sensitive file untrusted             | Sensitive file opened for reading by non-trusted program     | `WARNING`  | `cat /etc/shadow`            |
| Drop and execute new binary in container  | Executing binary not part of base image                      | `CRITICAL` | `/tmp/scan /`                |

The `exe_flags` field shows where a binary comes from. `bash` has `EXE_LOWER_LAYER`: it's part of the image. `/tmp/scan` has `EXE_UPPER_LAYER`: it's in the writable layer of the container. Someone added it after the container started.

The change to `index.html` has no alert. The stable rules don't cover writes to a web root.

---

## Step 4: Write a Custom Rule

A custom rule fills the gap from Step 3. The Helm chart writes it to `/etc/falco/rules.d/` in the Falco Pods.

The values file adds the rule:

```yaml
customRules:
  web-root-rules.yaml: |-
    - rule: Write to nginx web root
      desc: A process opened a file in the nginx web root for writing
      condition: >
        open_write and container
        and fd.name startswith /usr/share/nginx/html/
      output: >
        File in the nginx web root opened for writing
        (file=%fd.name command=%proc.cmdline user=%user.name
        container=%container.name pod=%k8s.pod.name ns=%k8s.ns.name)
      priority: WARNING
      tags: [filesystem, mitre_persistence]
```
- `open_write`: A macro for a file opened for writing.
- `container`: A macro that skips processes on the host.
- `fd.name`: The full path of the file.
- `%proc.cmdline`: The command with its arguments.
- `%k8s.pod.name` and `%k8s.ns.name`: The Pod and the Namespace.

Save the file as `falco-values.yaml`.

**Upgrade Falco with the custom rule**
```bash
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --reuse-values \
  -f falco-values.yaml
```

**Wait until the Falco Pods are back**
```bash
kubectl -n falco rollout status daemonset/falco
```

The Falco Pods restart with new names. Save the name again.

**Save the name of the Falco Pod on the worker node**
```bash
FALCO_POD=$(kubectl -n falco get pods --field-selector spec.nodeName=k8s-worker-1 -o name)
```

**Change the web page again**
```bash
kubectl -n runtime exec web -- sh -c 'echo hacked > /usr/share/nginx/html/index.html'
```

**Show the alert of the custom rule**
```bash
kubectl -n falco logs $FALCO_POD -c falco | grep "web root"
```
```
21:51:13.313626335: Warning File in the nginx web root opened for writing (file=/usr/share/nginx/html/index.html command=sh -c echo hacked > /usr/share/nginx/html/index.html user=root container=web pod=web ns=runtime) container_id=795c4790cf1d container_name=web container_image_repository=docker.io/library/nginx container_image_tag=latest k8s_pod_name=web k8s_ns_name=runtime
```

The alert shows the file, the command, the Pod, and the Namespace.

---

## Step 5: Make the Container Immutable

Falco detects the attack, but doesn't stop it. An immutable container stops part of it: with a read-only root file system, nobody can change the web page or drop a binary in the image.

nginx needs a few writable directories for its cache and its PID file. Each one gets an `emptyDir` volume:

**Create the manifest**
```bash
cat <<'EOF' > web-ro.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-ro
  namespace: runtime
spec:
  nodeName: k8s-worker-1
  containers:
    - name: nginx
      image: nginx
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
      volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
  volumes:
    - name: cache
      emptyDir: {}
    - name: run
      emptyDir: {}
EOF
```
- `readOnlyRootFilesystem: true`: Mounts the image file system read-only.
- `allowPrivilegeEscalation: false`: A process can't get more rights than its parent.
- `emptyDir`: A writable directory for each path that nginx writes to.

> **_Note:_** The `emptyDir` paths stay writable. An attacker can still drop a binary in `/var/run` or `/var/cache/nginx` and run it.

**Create the immutable Pod**
```bash
kubectl apply -f web-ro.yaml
```

**Check that nginx runs**
```bash
kubectl -n runtime get pod web-ro
```
```
NAME     READY   STATUS    RESTARTS   AGE
web-ro   1/1     Running   0          13s
```

**Try to change the web page**
```bash
kubectl -n runtime exec web-ro -- sh -c 'echo hacked > /usr/share/nginx/html/index.html'
```
```
sh: 1: cannot create /usr/share/nginx/html/index.html: Read-only file system
command terminated with exit code 2
```

**Try to drop a binary**
```bash
kubectl -n runtime exec web-ro -- cp /bin/ls /usr/local/bin/scan
```
```
cp: cannot create regular file '/usr/local/bin/scan': Read-only file system
command terminated with exit code 1
```

Both commands fail with a read-only file system. The attacker can still read `/etc/shadow`, and Falco still reports it.

> **_Warning:_** An immutable container doesn't stop `kubectl exec`. Remove the `pods/exec` permission from users who don't need it, as in [RBAC Users](/articles/kubernetes/cks/2-rbac-users/).

---

## Step 6: Find the Pod From an Alert

An alert often shows only a container ID, and you need to know which Pod it belongs to. `crictl` on the node finds the Pod.

**Show the container ID in the alerts**
```bash
kubectl -n falco logs $FALCO_POD -c falco | grep -o 'container_id=[a-z0-9]*' | sort -u
```
```
container_id=795c4790cf1d
container_id=b781920d4b99
```

The second ID belongs to `longhorn-manager`, the storage system on the node. Falco reports alerts for every container on the node, not only for the attack. The next commands look up the first ID.

**Find the container on `k8s-worker-1`**
```bash
sudo crictl ps --id 795c4790cf1d
```
```
CONTAINER       IMAGE           STATE     NAME  POD ID          POD   NAMESPACE
795c4790cf1d9   14fb90841494b   Running   web   849411dcd64c9   web   runtime
```

The `POD` and `NAMESPACE` columns already show the Pod. Older `crictl` versions don't have these columns. Use `crictl inspect` instead.

**Show the Pod of the container**
```bash
sudo crictl inspect 795c4790cf1d | jq -r '.status.labels["io.kubernetes.pod.name"], .status.labels["io.kubernetes.pod.namespace"]'
```
```
web
runtime
```

---

## Cleanup

In my homelab, I keep Falco running. I only delete the test Pods.

**Delete the test Namespace**
```bash
kubectl delete namespace runtime
```

To remove Falco as well, uninstall the chart.

**Uninstall Falco**
```bash
helm uninstall falco -n falco
```

---

## Common Errors

| Error                                                | Cause                                                            | Fix                                                                      |
| ---------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------ |
| A Falco Pod stays in `CrashLoopBackOff`              | The modern eBPF driver needs kernel 5.8 or later with BTF        | Check the kernel version with `uname -r`, or use `driver.kind=kmod`      |
| No alerts appear                                     | The logs of the wrong Falco Pod                                  | Read the Pod on the node where the target Pod runs                       |
| Alerts appear late                                   | `tty` is off, and the output is buffered                         | Set `tty=true`                                                           |
| No "Terminal shell in container" alert               | `kubectl exec` without `-it`, no terminal                        | Use `kubectl exec -it`                                                   |
| The custom rule doesn't load                         | A YAML or condition error in the rule                            | Read the Falco log at startup for the rule error                         |
| nginx doesn't start with a read-only root            | nginx can't write its cache or PID file                          | Add `emptyDir` volumes for `/var/cache/nginx` and `/var/run`             |

---

## Conclusion

Falco closes the gap that the other CKS projects leave open. Admission policies check a Pod before it starts. Audit logs record the requests to the API server. Falco watches what the processes do inside the containers, after they start.

Falco detects, it doesn't block. An immutable container blocks part of the attack: no new binaries in the image, no changed files. Together, they make an attack harder and leave a trace when it happens.

For production, send the alerts to a central system with Falcosidekick, for example to Slack or Loki. An alert in a Pod log is lost when the Pod is deleted.

---

## Reference

| Concept                              | What it means                                                         |
| ------------------------------------ | --------------------------------------------------------------------- |
| Falco                                | Watches system calls on the node and alerts on matching rules         |
| `modern_ebpf`                        | The eBPF driver built into Falco                                      |
| Rule                                 | `rule`, `desc`, `condition`, `output`, and `priority`                 |
| Macro                                | A reusable part of a condition, such as `open_write` or `container`   |
| `customRules`                        | The Helm value for custom rules files in `/etc/falco/rules.d/`        |
| `readOnlyRootFilesystem`             | Mounts the container image file system read-only                      |
| `/etc/falco/falco_rules.local.yaml`  | The file for local rules on a host install                            |

- [Falco: Deploy on Kubernetes](https://falco.org/docs/setup/kubernetes/)
- [Falco: Default and Local Rules](https://falco.org/docs/concepts/rules/default-custom/)
- [Falco: Supported Fields](https://falco.org/docs/reference/rules/supported-fields/)
- [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

---

## Best Practices

- Run Falco on every node, including the control plane nodes.
- Keep the default rules up to date, and put custom rules in separate files.
- Send the alerts to a central system.
- Run containers with `readOnlyRootFilesystem: true` and `emptyDir` volumes for the writable paths.
- Set `allowPrivilegeEscalation: false`.
- Limit `pods/exec` with RBAC.

---

## CKS Exam Takeaways

- On the exam, Falco often runs on the node as a service. Read the alerts with `journalctl -u falco-modern-bpf` or `journalctl -u falco-kmod`, depending on the driver, or in the file from `/etc/falco/falco.yaml`.
- Falco loads `/etc/falco/falco_rules.yaml`, then `/etc/falco/falco_rules.local.yaml`, then `/etc/falco/rules.d/`. Put changes in the local file.
- A rule needs `rule`, `desc`, `condition`, `output`, and `priority`.
- To change the output of a rule, copy it to the local file with the same name and change the `output`.
- Find the Pod of a container ID with `crictl ps --id` and `crictl inspect`.
- `readOnlyRootFilesystem: true` makes a container immutable. Add `emptyDir` volumes for the paths the application writes to.

---
