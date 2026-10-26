---
title: kube-bench
date: 2026-10-24
weight: 1
tags:
  - Security
  - CKS
  - CIS Benchmarks
  - kube-bench
  - kubeadm
  - Tutorial
---
# Project kube-bench

In this project, I run [kube-bench](https://github.com/aquasecurity/kube-bench) as a Job on my homelab cluster, read the CIS Kubernetes Benchmark document, and fix a few recommendations. I start with the first control plane node on my HA cluster and finish the other two control plane nodes with kubeadm.

kube-bench is an open-source tool from Aqua Security that checks a Kubernetes cluster against the CIS Kubernetes Benchmark. It inspects the configuration of the control plane and worker node components, reports which settings don't meet the hardening recommendations, and prints a remediation for each finding.

---
## CIS Kubernetes Benchmarks Document

I use the CIS Kubernetes Benchmark document to understand each recommendation and follow its **Remediation** to fix it.

### How to Download the Benchmark

The benchmark is a free PDF from the CIS website.

1. Go to [**CIS Kubernetes Benchmarks**](https://www.cisecurity.org/cis-benchmarks/#kubernetes).
2. Go to the `Server Software` section and click on `Virtualization`.
3. This category lists multiple items; search for Kubernetes.
4. Expand Kubernetes and click the download link.

---

## As a Job

The [kube-bench repository](https://github.com/aquasecurity/kube-bench) has ready-to-use Job manifests:

- [**Control plane node manifest**](https://github.com/aquasecurity/kube-bench/blob/main/job-master.yaml)
- [**Worker node manifest**](https://github.com/aquasecurity/kube-bench/blob/main/job-node.yaml)

| Manifest          | Inspects             |
|-------------------|----------------------|
| `job-master.yaml` | A control plane node |
| `job-node.yaml`   | A worker node        |

The original `job-master.yaml` uses a `nodeSelector` and a toleration to schedule the Pod on a tainted control plane node. In my homelab, the control plane nodes are untainted because they run workloads. I comment out both settings and pin the Job with `nodeName`.

**Copy the raw YAML from GitHub**
```text
https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml
```

**Create a Job manifest with the YAML**
```bash
vim kube-bench-master-1-job.yaml
```

**Modify the manifest to run on the control plane node**
```yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-master-1
spec:
  template:
    spec:
      hostPID: true # host processes
      nodeName: k8s-master-1 # node the Job runs on
      #...
```
- `hostPID: true`: Required to check the running host's processes.
- `nodeName`: The node the Job runs on.

> **_Note:_** In production, the control plane nodes stay tainted and don't run workloads.

**Apply the Job manifest**
```bash
kubectl apply -f kube-bench-master-1-job.yaml
```

**Wait until the Job completes**
```bash
kubectl get job kube-bench-master-1 -w
```
- `-w`: Watches the Job's status until it completes.

**Read the logs for the result**
```bash
kubectl logs job/kube-bench-master-1
```
```
...
== Summary master ==
41 checks PASS
10 checks FAIL
9 checks WARN
0 checks INFO

== Summary total ==
41 checks PASS
10 checks FAIL
9 checks WARN
0 checks INFO
```

---

## Reading the Report

**kube-bench** runs the CIS Kubernetes Benchmark against the cluster and reports how the control plane component is configured compared to the hardening recommendations.

**View the Job logs**
```bash
kubectl logs job/kube-bench-master-1
```
```
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
...
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
...
[INFO] 1.2 API Server
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
[PASS] 1.2.2 Ensure that the --token-auth-file parameter is not set (Automated)
...
[WARN] 1.2.11 Ensure that the admission control plugin AlwaysPullImages is set (Manual)
...
[FAIL] 1.2.15 Ensure that the --profiling argument is set to false (Automated)
[FAIL] 1.2.16 Ensure that the --audit-log-path argument is set (Automated)
[FAIL] 1.2.17 Ensure that the --audit-log-maxage argument is set to 30 or as appropriate (Automated)
[FAIL] 1.2.18 Ensure that the --audit-log-maxbackup argument is set to 10 or as appropriate (Automated)
[FAIL] 1.2.19 Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)
...
[INFO] 1.3 Controller Manager
...
[FAIL] 1.3.2 Ensure that the --profiling argument is set to false (Automated)
...
[INFO] 1.4 Scheduler
[FAIL] 1.4.1 Ensure that the --profiling argument is set to false (Automated)
[PASS] 1.4.2 Ensure that the --bind-address argument is set to 127.0.0.1 (Automated)

== Remediations master ==
1.1.12 On the etcd server node, get the etcd data directory, passed as an argument --data-dir,
from the command 'ps -ef | grep etcd'.
Run the below command (based on the etcd data directory found above).
For example, chown etcd:etcd /var/lib/etcd

1.2.1 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the control plane node and set the below parameter.
--anonymous-auth=false

1.2.11 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the control plane node and set the --enable-admission-plugins parameter to include
AlwaysPullImages.
--enable-admission-plugins=...,AlwaysPullImages,...

1.2.15 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the control plane node and set the below parameter.
--profiling=false

1.2.16 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the control plane node and set the --audit-log-path parameter to a suitable path and
file where you would like audit logs to be written, for example,
--audit-log-path=/var/log/apiserver/audit.log

1.2.17 to 1.2.19 Same file: --audit-log-maxage=30, --audit-log-maxbackup=10, --audit-log-maxsize=100

1.3.2 / 1.4.1 Edit kube-controller-manager.yaml / kube-scheduler.yaml and set --profiling=false
...

== Summary master ==
41 checks PASS
10 checks FAIL
9 checks WARN
0 checks INFO
```

The report above only shows the findings I fixed in this project.

### The Structure

kube-bench groups the checks by component. Section 1 covers the control plane: the node config files (1.1), API server (1.2), controller manager (1.3), and scheduler (1.4). Section 2 covers `etcd`. Section 3 covers control plane configuration: authentication, authorization, and logging. Section 4 covers the worker nodes, and Section 5 covers policies.

### The Four Statuses

Every line in the report starts with one of four statuses.

| Status       | Meaning                                                        | What you do                     |
|--------------|----------------------------------------------------------------|---------------------------------|
| **`[PASS]`** | Meets the benchmark                                            | Nothing                         |
| **`[FAIL]`** | Checked automatically and **doesn't** meet the benchmark       | Fix it with the remediation     |
| **`[WARN]`** | kube-bench cannot verify by itself; a manual check is required | Read the remediation and decide |
| **`[INFO]`** | Headings and information                                       | Nothing                         |

### What the Findings Mean

The job on `k8s-master-1` reports these findings, each linked to its fix.

- [**1.1.12**](#1112-etcd-data-directory-ownership) The etcd data directory (`/var/lib/etcd`) is not owned by the `etcd:etcd` user. Fix with `chown etcd:etcd /var/lib/etcd`.
- [**1.2.1**](#121-anonymous-requests) The API server accepts anonymous requests on every path. RBAC is the only thing that stops them.
- [**1.2.11**](#1211-alwayspullimages) Without the `AlwaysPullImages` admission plugin, a Pod can reuse an image cached on the node without a credential check.
- [**1.2.15 / 1.3.2 / 1.4.1**](#1215--132--141-profiling) Profiling is enabled on the API server, controller manager, and scheduler. Set `--profiling=false` on each.
- [**1.2.16 to 1.2.19**](#1216-to-1219-audit-logging) API server audit logging is not configured: no audit path and no retention policy. Without audit logs, you can't see who did what in the cluster. Set `--audit-log-path` and the `--audit-log-maxage`, `--audit-log-maxbackup`, and `--audit-log-maxsize` flags.

---

## The Fix Pattern

On a kubeadm cluster, the API server, controller manager, scheduler, and etcd are **static Pods**. Their manifests are in `/etc/kubernetes/manifests/` on the control plane node. You add a flag under the container's `command` list. The kubelet watches this directory and restarts the static Pod automatically when a file changes. While the API server restarts, `kubectl` doesn't respond for a while.

| What you change     | Where                              | How it applies                  | Checks              |
|---------------------|------------------------------------|---------------------------------|---------------------|
| Control plane flags | `/etc/kubernetes/manifests/*.yaml` | kubelet restarts the static Pod | 1.2.x, 1.3.x, 1.4.x |
| Host files          | the file itself                    | immediately                     | 1.1.x               |

> **_Note:_** `kubeadm upgrade` regenerates the manifests in `/etc/kubernetes/manifests/` from the `ClusterConfiguration` in the `kubeadm-config` ConfigMap. Every flag and volume you add by hand is lost. To keep them, also add them under `apiServer`, `controllerManager`, or `scheduler` as `extraArgs` and `extraVolumes`. See [Persist and Roll Out the Fixes](#persist-and-roll-out-the-fixes).

### Back Up the Manifest Files

> **_Note:_** Always back up the files **outside** the manifests directory.

```bash
sudo mkdir -p /etc/kubernetes/backups
for f in etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml; do
  sudo cp /etc/kubernetes/manifests/$f \
    /etc/kubernetes/backups/$f.$(date +%Y-%m-%d)-backup
done
```

> **_Note:_** Everything in the `manifests` directory will be started by the kubelet.

### Re-run the kube-bench Job

After every fix, run the Job again to check the result. An existing Job can't run again. Delete it first, and apply the manifest again.

**SSH into the node and edit the manifest file**
```bash
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Wait until the static Pods on k8s-master-1 are running again**
```bash
kubectl -n kube-system get pods --field-selector spec.nodeName=k8s-master-1
```

**Delete the old Job**
```bash
kubectl delete job kube-bench-master-1
```

**Re-apply the Job manifest and wait until it completes**
```bash
kubectl apply -f kube-bench-master-1-job.yaml
kubectl wait --for=condition=complete job/kube-bench-master-1
```

**View the result in the logs**
```bash
kubectl logs job/kube-bench-master-1 | grep <CHECK_NUMBER>
```

---

## The Fixes

All examples below are from the kube-bench Job on the control plane node. The fixes follow the report's order. Check numbers can change between CIS Benchmark versions. If a number doesn't match the report, search for the flag name instead.

### 1.1.12 `etcd` Data Directory Ownership

This indicates that the host directory storing the cluster's state data is accessible by an incorrect Linux user or group. The **CIS Kubernetes Benchmark** enforces this check because any unauthorized user who has access to the `etcd` data directory can bypass Kubernetes RBAC entirely, read Secrets, or corrupt the entire cluster state.

> [FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
> 
> 1.1.12 On the etcd server node, get the etcd data directory, passed as an argument --data-dir,  
> from the command 'ps -ef | grep etcd'.  
> Run the below command (based on the etcd data directory found above).  
> For example, chown etcd:etcd /var/lib/etcd

Follow the remediation: first find the real data directory, then change its owner.

**1. Find the current etcd data directory path**
```bash
ps -ef | grep etcd | grep -- '--data-dir'
# --data-dir=/var/lib/etcd
```
This finds the running `etcd` process and prints its `--data-dir` argument.

**2. Check if the 'etcd' user exists; if not, create it as a system user**
```bash
id etcd || sudo useradd -r -s /sbin/nologin -M etcd && id etcd
# uid=999(etcd) gid=990(etcd) groups=990(etcd)
```
- `||`: Runs the right side only (`sudo ...`) if the left side failed.
- `-r`: Gets a UID from the low system range with no aging info.
- `-s /sbin/nologin`: Without a login shell, normal for a service user.
- `-M`: Doesn't create a home directory. Services don't need one.

**3. Ensure the `etcd` user and group own the data directory recursively**
```bash
sudo chown -R etcd:etcd /var/lib/etcd
```
- `-R`: Recursive, includes everything inside `/var/lib/etcd`.

**4. Verify the permissions, user ownership, and group ownership of the directory**
```bash
stat -c '%a %U:%G' /var/lib/etcd
# 700 etcd:etcd
```

> **_Note:_** On a kubeadm cluster, `etcd` runs as root. The new owner passes the check without changing how etcd runs.

**5. Re-run the Job and view the logs**
```bash
kubectl delete job kube-bench-master-1
kubectl apply -f kube-bench-master-1-job.yaml
kubectl wait --for=condition=complete job/kube-bench-master-1
kubectl logs job/kube-bench-master-1 | grep '1.1.12'
```
```text
[PASS] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
```

---

### 1.2.1 Anonymous Requests

Anonymous access lets **unauthenticated users** interact with the API server.

> [WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
> 
> 1.2.1 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml  
> on the control plane node and set the below parameter.  
> --anonymous-auth=false  

**1. Check the current state**
```bash
kubectl -n kube-system get pod kube-apiserver-k8s-master-1 -o yaml | \
  grep -- '--anonymous-auth'
```
If you see `--anonymous-auth=true` (default), or nothing in the terminal, anonymous requests are allowed.

> **_Warning:_** Steps 2 to 4 show why the remediation doesn't work on kubeadm. Don't apply them and go to step 5.

**2. Add the flag to the `/etc/kubernetes/manifests/kube-apiserver.yaml` manifest**
```yaml
apiVersion: v1
kind: Pod
#...
spec:
  containers:
  - command:
    - kube-apiserver
    - #...
    - --anonymous-auth=false
  image: #...
```

**3. Wait until the Pod is ready**
```bash
kubectl get pods -n kube-system --field-selector spec.nodeName=k8s-master-1 -w
```
```
NAME                                   READY   STATUS    RESTARTS       AGE
kube-apiserver-k8s-master-1            0/1     Running   1              100s
```
You will notice that the Pod is not ready because the API server isn't ready.

**4. Check the Pod events**
```bash
kubectl -n kube-system describe pod kube-apiserver-k8s-master-1 | grep -iA10 events
```
```
Events:
  Type     Reason     Message
  ----     ------     -------
  #...
  Warning  Unhealthy  spec.containers{kube-apiserver}: Startup probe failed: HTTP probe failed with statuscode: 401
```
Here you see the Pod is unhealthy because the startup probe can't access the API server without credentials. With `--anonymous-auth=false`, these probes get a 401, and the kubelet keeps restarting the API server.

On Kubernetes `v1.34` and later, you can allow anonymous access only for the health endpoints with an [`AuthenticationConfiguration`](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-authenticator-configuration) file. This replaces the `--anonymous-auth` flag, but **you can't set both**.

**5. Create `/etc/kubernetes/apiserver-probes-auth-config.yaml`**
```yaml
# Allow anonymous access to the kube-apiserver endpoints for probes
apiVersion: apiserver.config.k8s.io/v1
kind: AuthenticationConfiguration
anonymous:
  enabled: true
  conditions:
  - path: /livez
  - path: /readyz
```
- `/livez` and `/readyz`: Used by the kubelet probes.
- The `/healthz` endpoint is deprecated since `v1.16`.

> **_Note:_** `kubeadm join --discovery-token` reads the `cluster-info` ConfigMap anonymously. Joining a new node fails with this config. In my homelab, I don't join new nodes, and I leave it out. To join one, use `--discovery-file` with `kubeadm join`, or add `- path: /api/v1/namespaces/kube-public/configmaps/cluster-info` for the join.

**6. Pass the file to the API server and mount it**

The API server refuses to start with both flags. Remove `--anonymous-auth=false` before adding `--authentication-config`.

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - #...
    # CIS 1.2.1 Limit anonymous requests to /livez and /readyz,
    # kube-bench still reports [WARN] because it only checks --anonymous-auth
    - --authentication-config=/etc/kubernetes/apiserver-probes-auth-config.yaml
    volumeMounts:
    #...
    - name: probes-auth-config
      mountPath: /etc/kubernetes/apiserver-probes-auth-config.yaml
      readOnly: true
  volumes:
  #...
  - name: probes-auth-config
    hostPath:
      path: /etc/kubernetes/apiserver-probes-auth-config.yaml
      type: File
```

**7. Wait until the Pod is ready**
```bash
kubectl get pods -n kube-system --field-selector spec.nodeName=k8s-master-1 -w
```

**8. Re-run the Job and view the logs**
```bash
kubectl delete job kube-bench-master-1
kubectl apply -f kube-bench-master-1-job.yaml
kubectl wait --for=condition=complete job/kube-bench-master-1
kubectl logs job/kube-bench-master-1 | grep '1.2.1 '
```
```
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
```
kube-bench keeps reporting `[WARN]` because the check only looks for the `--anonymous-auth=false` flag, and you can't set that flag together with `--authentication-config`. Document it as an accepted risk.

**9. Verify authentication configuration endpoints**
```bash
curl -k https://192.168.3.110:6443/readyz
# ok
curl -k https://192.168.3.110:6443/api
# 401 Unauthorized
```
The API server accepts anonymous requests on `/readyz` and rejects them on all other paths.

#### HAProxy Health Check

The HAProxy load balancer in my homelab checks the API servers on the three control plane nodes without credentials. After this change, the `/healthz` check got a 401 and marked the control plane as down. Pointing the check to `/readyz` fixed this.

---

### 1.2.11 `AlwaysPullImages`

Without `AlwaysPullImages`, any user can run a Pod using an image already pulled to a node by knowing its name, without any authorization check for the image's ownership.

> [WARN] 1.2.11 Ensure that the admission control plugin AlwaysPullImages is set (Manual)
>
> 1.2.11 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml  
> on the control plane node and set the --enable-admission-plugins parameter to include  
> AlwaysPullImages.  
> --enable-admission-plugins=...,AlwaysPullImages,...  

**1. Verify that the argument exists**
```bash
ps -ef | grep kube-apiserver | grep -o -- '--enable-admission-plugins=[^ ]*'
# --enable-admission-plugins=NodeRestriction
```
- `ps -ef`: Lists every running process on the host machine.

Add `AlwaysPullImages` to the existing `--enable-admission-plugins` line, separated by a comma.

**2. Add AlwaysPullImages to the admission plugins**
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - #...
    # CIS 1.2.11 Ensure that the admission control plugin AlwaysPullImages is set
    - --enable-admission-plugins=NodeRestriction,AlwaysPullImages
```
`AlwaysPullImages` sets `imagePullPolicy: Always` in the Pod object. Before a container starts, the kubelet checks the image in the registry with the Pod's pull credentials. A Pod without valid credentials can't use an image that is cached on the node.

**3. Wait until the Pod is ready**
```bash
kubectl get pods -n kube-system --field-selector spec.nodeName=k8s-master-1 -w
```

**4. Re-run the Job and view the logs**
```bash
kubectl delete job kube-bench-master-1
kubectl apply -f kube-bench-master-1-job.yaml
kubectl wait --for=condition=complete job/kube-bench-master-1
kubectl logs job/kube-bench-master-1 | grep '1.2.11'
```
```
[PASS] 1.2.11 Ensure that the admission control plugin AlwaysPullImages is set (Manual)
```

**5. Verify that the argument exists**
```bash
ps -ef | grep kube-apiserver | grep -- '--enable-admission-plugins'
# --enable-admission-plugins=NodeRestriction,AlwaysPullImages
```

---

### 1.2.15 / 1.3.2 / 1.4.1 Profiling

With profiling enabled, a component exposes performance data that may contain sensitive information and can be used by an attacker to reveal system and program details.

> [FAIL] 1.2.15 Ensure that the --profiling argument is set to false (Automated)  
> [FAIL] 1.3.2 Ensure that the --profiling argument is set to false (Automated)  
> [FAIL] 1.4.1 Ensure that the --profiling argument is set to false (Automated)  
>
> 1.2.15 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml  
> on the control plane node and set the below parameter.  
> --profiling=false
> 
> 1.3.2 Edit the Controller Manager pod specification file /etc/kubernetes/manifests/kube-controller-manager.yaml  
> on the control plane node and set the below parameter.  
> --profiling=false
> 
> 1.4.1 Edit the Scheduler pod specification file /etc/kubernetes/manifests/kube-scheduler.yaml file  
> on the control plane node and set the below parameter.  
> --profiling=false

The same flag failed in all three manifests. Add it to the `command` list of each:

| Check  | File                                                     |
|--------|----------------------------------------------------------|
| 1.2.15 | `/etc/kubernetes/manifests/kube-apiserver.yaml`          |
| 1.3.2  | `/etc/kubernetes/manifests/kube-controller-manager.yaml` |
| 1.4.1  | `/etc/kubernetes/manifests/kube-scheduler.yaml`          |

**1. Verify that the argument exists**
```bash
ps -ef | grep 'kube-apiserver' | grep -- '--profiling'
ps -ef | grep 'kube-controller-manager' | grep -- '--profiling'
ps -ef | grep 'kube-scheduler' | grep -- '--profiling'
```
If the argument doesn't exist, add the flag.

**2. Add the `profiling` flag to the three manifests**
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - #...
    # CIS 1.2.15 Ensure that the --profiling argument is set to false
    - --profiling=false
```
```yaml
spec:
  containers:
  - command:
    - kube-controller-manager
    - #...
    # CIS 1.3.2 Ensure that the --profiling argument is set to false
    - --profiling=false
```
```yaml
spec:
  containers:
  - command:
    - kube-scheduler
    - #...
    # CIS 1.4.1 Ensure that the --profiling argument is set to false
    - --profiling=false
```

**3. Wait until the Pods are ready**
```bash
kubectl get pods -n kube-system --field-selector spec.nodeName=k8s-master-1 -w
```

**4. Re-run the Job and view the logs**
```bash
kubectl delete job kube-bench-master-1
kubectl apply -f kube-bench-master-1-job.yaml
kubectl wait --for=condition=complete job/kube-bench-master-1
kubectl logs job/kube-bench-master-1 | grep -E '1\.(2\.15|3\.2|4\.1) '
```
```
[PASS] 1.2.15 Ensure that the --profiling argument is set to false (Automated)
[PASS] 1.3.2 Ensure that the --profiling argument is set to false (Automated)
[PASS] 1.4.1 Ensure that the --profiling argument is set to false (Automated)
```

---

### 1.2.16 to 1.2.19 Audit Logging

The API server runs in a container. The log directory must be **mounted from the host**. Without the mount, the logs remain inside the container, are not visible on the host, and are lost when the container restarts.

> [FAIL] 1.2.16 Ensure that the --audit-log-path argument is set (Automated)  
> [FAIL] 1.2.17 Ensure that the --audit-log-maxage argument is set to 30 or as appropriate (Automated)  
> [FAIL] 1.2.18 Ensure that the --audit-log-maxbackup argument is set to 10 or as appropriate (Automated)  
> [FAIL] 1.2.19 Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)  
>
> 1.2.16 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml  
> on the control plane node and set the --audit-log-path parameter to a suitable path and  
> file where you would like audit logs to be written, for example,  
> --audit-log-path=/var/log/apiserver/audit.log  
> 
> 1.2.17 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml  
> on the control plane node and set the --audit-log-maxage parameter to 30  
> or as an appropriate number of days, for example,  
> --audit-log-maxage=30  
> 
> 1.2.18 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml  
> on the control plane node and set the --audit-log-maxbackup parameter to 10 or to an appropriate  
> value. For example,  
> --audit-log-maxbackup=10
> 
> 1.2.19 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml  
> on the control plane node and set the --audit-log-maxsize parameter to an appropriate size in MB.  
> For example, to set it as 100 MB, --audit-log-maxsize=100

All four recommendations are flags in the kube-apiserver manifest. The `--audit-log-path` flag uses an audit log file, which also needs a volume for the log directory.

**1. Verify that the arguments exist**
```bash
ps -ef | grep kube-apiserver | grep -o -- '--audit-log-[^ ]*'
```
If you see nothing in the terminal, the flags are not set in the kube-apiserver manifest.

**2. Add the four flags and mount the log directory**
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - #...
    # CIS 1.2.16 to 1.2.19 Ensure that the --audit-log-* arguments are set
    - --audit-log-path=/var/log/apiserver/audit.log
    - --audit-log-maxage=30     # days to keep old files
    - --audit-log-maxbackup=10  # number of old files to keep
    - --audit-log-maxsize=100   # MB before the file is rotated
    volumeMounts:
    #...
    # CIS 1.2.16 Ensure that the --audit-log-path argument is set
    - name: audit-log
      mountPath: /var/log/apiserver
  volumes:
  #...
  # CIS 1.2.16 Ensure that the --audit-log-path argument is set
  - name: audit-log
    hostPath:
      path: /var/log/apiserver
      type: DirectoryOrCreate
```

> **_Note:_** This passes the four checks, but nothing gets logged yet: the API server only writes audit events once an audit policy is set with the `--audit-policy-file` flag.

**3. Wait until the Pod is ready**
```bash
kubectl get pods -n kube-system --field-selector spec.nodeName=k8s-master-1 -w
```

**4. Re-run the Job and view the logs**
```bash
kubectl delete job kube-bench-master-1
kubectl apply -f kube-bench-master-1-job.yaml
kubectl wait --for=condition=complete job/kube-bench-master-1
kubectl logs job/kube-bench-master-1 | grep -E '1\.2\.1[6-9]'
```
```
[PASS] 1.2.16 Ensure that the --audit-log-path argument is set (Automated)
[PASS] 1.2.17 Ensure that the --audit-log-maxage argument is set to 30 or as appropriate (Automated)
[PASS] 1.2.18 Ensure that the --audit-log-maxbackup argument is set to 10 or as appropriate (Automated)
[PASS] 1.2.19 Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)
```

---

## Persist and Roll Out the Fixes

Every fix in this project lives in the static Pod manifests in `/etc/kubernetes/manifests/`. Those files are **not** the source of truth on a kubeadm cluster. When kubeadm upgrades the nodes, it regenerates them from the `ClusterConfiguration` in the `kubeadm-config` ConfigMap, and it removes any flags and volumes you added by hand. The next kube-bench run reports the same findings again.

To keep the fixes persistent, add them to `kubeadm-config` as well. kubeadm then writes them into the manifests on every upgrade, on every control plane node.

**1. Back up the ConfigMap**
```bash
kubectl -n kube-system get cm kubeadm-config -o yaml | \
  sudo tee /etc/kubernetes/backups/kubeadm-config.yaml.$(date +%Y-%m-%d)-backup
```

**2. View the current `ClusterConfiguration`**
```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'
```

**3. Add the fixes**
```bash
kubectl -n kube-system edit cm kubeadm-config
```

Copy/paste the recommendations from the fixes and add them to the `extraArgs` and `extraVolumes` under `apiServer`, `controllerManager`, and `scheduler`. Keep the fields that are already there.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
apiServer:
  extraArgs:
  # CIS 1.2.1
  - name: authentication-config
    value: /etc/kubernetes/apiserver-probes-auth-config.yaml
  # CIS 1.2.11
  - name: enable-admission-plugins
    value: NodeRestriction,AlwaysPullImages
  # CIS 1.2.15
  - name: profiling
    value: "false"
  # CIS 1.2.16 to 1.2.19
  - name: audit-log-path
    value: /var/log/apiserver/audit.log
  - name: audit-log-maxage
    value: "30"
  - name: audit-log-maxbackup
    value: "10"
  - name: audit-log-maxsize
    value: "100"

  extraVolumes:
  - name: probes-auth-config
    hostPath: /etc/kubernetes/apiserver-probes-auth-config.yaml
    mountPath: /etc/kubernetes/apiserver-probes-auth-config.yaml
    readOnly: true
    pathType: File
  - name: audit-log
    hostPath: /var/log/apiserver
    mountPath: /var/log/apiserver
    readOnly: false
    pathType: DirectoryOrCreate

controllerManager:
  extraArgs:
  # CIS 1.3.2
  - name: profiling
    value: "false"

scheduler:
  extraArgs:
  # CIS 1.4.1
  - name: profiling
    value: "false"
```
- `extraArgs`: Add the flag without the leading `--`.
- `value`: Always a string. Put numbers and booleans in double quotes.
- `extraVolumes`: Becomes both the `volumes` and `volumeMounts` entries in the manifest.

> **_Warning:_** In the `enable-admission-plugins`, **always** include the kubeadm defaults (`NodeRestriction`).

**4. Check what kubeadm would generate**
```bash
sudo kubeadm upgrade diff v1.35.5
```

This compares the current manifests with what kubeadm would generate for the next version update, without making any changes. If the output shows no lost flags or volumes, the ConfigMap matches the manifests.

> **_Note:_** kubeadm doesn't keep comments. After an upgrade, the `# CIS ...` comments in the manifests are gone, even though the flags stay. Keep your notes in this ConfigMap or in your docs instead.

**5. Keep the host files on every node**

`kubeadm-config` only covers flags and volumes. The two fixed recommendations below are on the node itself. Every control plane node needs them:

**[1.1.12 etcd data directory ownership](#1112-etcd-data-directory-ownership)**

- What: `/var/lib/etcd` owned by `etcd:etcd`
- Why: This is a host setting, not a flag. kubeadm never sets it.

**[1.2.1 Anonymous requests](#121-anonymous-requests)**

- What: `/etc/kubernetes/apiserver-probes-auth-config.yaml`
- Why: The volume uses `pathType: File`. The API server doesn't start without this file.

### Roll Out to the Other Control Plane Nodes

I use only one control plane node to run kube-bench with the fixes. The `kubeadm-config` ConfigMap contains these fixes for the entire cluster, but kubeadm writes the manifests only when you run it on the other nodes.

Instead of editing each manifest by hand, kubeadm regenerates them from the ConfigMap. kubeadm writes each node's own manifest, with its own `--advertise-address`. With this, you can't make a mistake on the other nodes. Do **one node at a time**. HAProxy always has healthy API servers.

### Prepare the Node

Before kubeadm regenerates the manifests, back up the manifests and add the required host files.

**1. SSH into the node and back up the manifests**

Store the backups outside `/etc/kubernetes/manifests/`. Otherwise, the kubelet starts them as extra Pods.

```bash
sudo mkdir -p /etc/kubernetes/backups
for f in etcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml; do
  sudo cp /etc/kubernetes/manifests/$f \
    /etc/kubernetes/backups/$f.$(date +%Y-%m-%d)-backup
done
ls -l /etc/kubernetes/backups/
```

> **_Note:_** kubeadm also saves the old manifests in `/etc/kubernetes/tmp/` when it regenerates them. Your own backup is still useful because it has a clear name and date.

**2. Create the authentication config file**
```bash
sudo vim /etc/kubernetes/apiserver-probes-auth-config.yaml
```
```yaml
# CIS 1.2.1 Allow anonymous access to the kube-apiserver endpoints for probes
apiVersion: apiserver.config.k8s.io/v1
kind: AuthenticationConfiguration
anonymous:
  enabled: true
  conditions:
    - path: /livez
    - path: /readyz
```

> **_Note:_** Without this file, the API server won't start.

### Update the Other Control Plane Nodes

kubeadm writes the manifests of the node from the `kubeadm-config` ConfigMap. Check the changes first, then apply them.

**1. See what would change**
```bash
sudo kubeadm upgrade diff v1.35.5
```

**2. Do a dry run of the regeneration**
```bash
sudo kubeadm upgrade node phase control-plane --dry-run --certificate-renewal=false --etcd-upgrade=false
```
This simulates everything and prints what would change without touching the node. Nothing is written to disk or applied.

**3. Regenerate the manifests**
```bash
sudo kubeadm upgrade node phase control-plane --certificate-renewal=false --etcd-upgrade=false
```
- kubeadm rewrites the static Pod manifests for the API server, controller manager, and scheduler in `/etc/kubernetes/manifests/` to the target version from the `kubeadm-config` ConfigMap. The kubelet detects the changed files and restarts only those Pods. A manifest without changes is skipped.
- `--certificate-renewal=false`: Doesn't renew the node's control plane certificates.
- `--etcd-upgrade=false`: Leaves the node's etcd static Pod manifest untouched.

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
W1024 13:07:19.676043 1017656 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" not found
[upgrade/control-plane] Upgrading your Static Pod-hosted control plane instance to version "v1.35.5"...
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests269968554"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-10-24-13-07-19/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Current and new manifests of kube-controller-manager are equal, skipping upgrade
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Current and new manifests of kube-scheduler are equal, skipping upgrade
[upgrade/control-plane] The control plane instance for this node was successfully upgraded!
```

> **_Note:_** The warning `configmaps "kube-proxy" not found` is expected in my homelab, because I replaced it with Cilium on the cluster.

**4. Verify the Pods are ready**
```bash
kubectl get pods -n kube-system --field-selector spec.nodeName=k8s-master-2 -w
```
```
NAME                                   READY   STATUS    RESTARTS       AGE
cilium-envoy-zgxmh                     1/1     Running   5 (110d ago)   212d
cilium-n7qcb                           1/1     Running   5 (110d ago)   212d
etcd-k8s-master-2                      1/1     Running   4 (110d ago)   135d
kube-apiserver-k8s-master-2            1/1     Running   0              115s
kube-controller-manager-k8s-master-2   1/1     Running   0              6d21h
kube-scheduler-k8s-master-2            1/1     Running   0              6d21h
```

**5. Do the CIS 1.1.12 host-level fix by hand**

Each node needs its own `etcd` user before it can own the data directory. See [1.1.12 etcd Data Directory Ownership](#1112-etcd-data-directory-ownership) for the full steps.

**Create the `etcd` system user if it doesn't exist**
```bash
id etcd || sudo useradd -r -s /sbin/nologin -M etcd
```

**Change the owner of the data directory**
```bash
sudo chown -R etcd:etcd /var/lib/etcd
```

**6. Run kube-bench on the node**

Create a kube-bench Job for the node and compare the report with `k8s-master-1`.

**Copy the Job manifest for k8s-master-2**
```bash
cp kube-bench-master-1-job.yaml kube-bench-master-2-job.yaml
```

**Change the `name` and `nodeName`** 
```yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-master-2 # name of the Job
spec:
  template:
    spec:
      #...
      nodeName: k8s-master-2 # node the Job runs on
      #...
```

**Apply the Job manifest and wait until it completes**
```bash
kubectl apply -f kube-bench-master-2-job.yaml
kubectl wait --for=condition=complete job/kube-bench-master-2
```

**Check the fixed recommendations in the logs**
```bash
kubectl logs job/kube-bench-master-2 | grep '1.1.12'
kubectl logs job/kube-bench-master-2 | grep '1.2.1 '
kubectl logs job/kube-bench-master-2 | grep '1.2.11'
kubectl logs job/kube-bench-master-2 | grep -E '1\.(2\.15|3\.2|4\.1) '
kubectl logs job/kube-bench-master-2 | grep -E '1\.2\.1[6-9]'
```
```
[PASS] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
1.2.1 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
[PASS] 1.2.11 Ensure that the admission control plugin AlwaysPullImages is set (Manual)
[PASS] 1.2.15 Ensure that the --profiling argument is set to false (Automated)
[PASS] 1.3.2 Ensure that the --profiling argument is set to false (Automated)
[PASS] 1.4.1 Ensure that the --profiling argument is set to false (Automated)
[PASS] 1.2.16 Ensure that the --audit-log-path argument is set (Automated)
[PASS] 1.2.17 Ensure that the --audit-log-maxage argument is set to 30 or as appropriate (Automated)
[PASS] 1.2.18 Ensure that the --audit-log-maxbackup argument is set to 10 or as appropriate (Automated)
[PASS] 1.2.19 Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)
```

On the HAProxy stats page, the node must be green before starting with the next node.

**If something goes wrong**, restore the backup. The kubelet then restarts the Pod with the old configuration.

```bash
sudo cp /etc/kubernetes/backups/kube-apiserver.yaml.<DATE>-backup \
  /etc/kubernetes/manifests/kube-apiserver.yaml
```

Then repeat all steps for `k8s-master-3`.

---

## Worker Nodes

This project only covers the control plane nodes. Worker node checks (CIS section 4) need the `job-node.yaml` Job.

---

## Troubleshoot the API Server

Editing static manifest files is a common way to break the cluster.

Before every change, [back up the manifest files](#back-up-the-manifest-files) outside the manifests directory.

If `kubectl` hangs or says "connection refused" for more than a minute or two, the API server didn't start. `kubectl` can't help you. Use the container runtime on the node. In my homelab, I installed containerd.

**Check the state of the API server container**
```bash
sudo crictl ps -a | grep kube-apiserver
```
- `-a`: Shows all containers. A status of `Exited` and the `ATTEMPT` count mean the API server keeps crashing.

**Read the logs of the API server container**
```bash
sudo crictl logs <CONTAINER_ID>
```
- `<CONTAINER_ID>`: The ID from the first column of `crictl ps -a`. 
- The `logs` show why the API server stopped.

**Read the API server logs on disk**
```bash
sudo tail -n 50 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log
```

**Read the kubelet logs**
```bash
sudo journalctl -u kubelet | tail -50
```

The kubelet logs show manifest errors, such as a YAML parse error. In that case, the kubelet never starts the container, and `crictl` shows nothing.

Most broken manifests come from one of three mistakes: a YAML indentation error, a typo in a flag name (the API server refuses unknown flags), or a broken comma-separated list such as `--enable-admission-plugins`.

If you can't find it quickly, restore the backup, wait for the API server, and redo the change carefully.

**Restore the backup of the manifest**
```bash
sudo cp /etc/kubernetes/backups/kube-apiserver.yaml.<DATE>-backup \
  /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Wait until the API server container is running again**
```bash
sudo crictl ps | grep kube-apiserver
```

---