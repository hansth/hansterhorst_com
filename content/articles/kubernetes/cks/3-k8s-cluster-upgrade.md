---
title: Kubernetes Cluster Upgrade
date: 2026-10-21
weight: 3
tags:
  - CKS
  - Kubernetes
  - kubeadm
  - Upgrade
  - Tutorial
---
# Project Kubernetes Cluster Upgrade

I installed my homelab cluster with **v1.32** on purpose, to practice cluster upgrades. I upgraded it one minor version at a time: `1.32 → 1.33 → 1.34 → 1.35`. Each upgrade follows the same steps, on every node.

---

## How the Upgrade Process Works

A cluster upgrade has two main phases:

1. Upgrading the three control plane nodes.
2. Upgrading the worker node.

### What Happens During a Control Plane Upgrade

In a cluster with a single control plane node, the kube-apiserver and the other control plane components are unavailable during the upgrade. The applications keep running: the Pods on the worker node stay up and reachable.

What does it mean:

- You cannot interact with the cluster via `kubectl`.
- The API server is unavailable.
- If a Pod is deleted or its node fails during the upgrade, it will not be replaced because the `kube-controller-manager` and `kube-scheduler` are also offline. A crashed container is still restarted by the kubelet on its node.

In my setup, I don't have this problem because I run three control plane nodes and upgrade them one at a time. While one node is being upgraded, the others remain active and fully operational.

---

## Which Components Get Upgraded

Kubernetes contains multiple components, and which ones you need to upgrade depends on the node type.

**Control plane nodes** run:

1. `kube-apiserver`
2. `kube-controller-manager`
3. `kube-scheduler`
4. `kubelet`
5. `kubectl`
6. `etcd`
7. CoreDNS
8. ~~`kube-proxy`~~ replaced with Cilium

The first five share the same Kubernetes release versioning. `etcd` and CoreDNS use their own versioning schemes. kubeadm installed them, and `kubeadm upgrade` upgrades them automatically. In my setup, I replaced kube-proxy with Cilium, installed Cilium separately, and upgrade it separately as well.

**Worker nodes** run:

- `kubelet`
- ~~`kube-proxy`~~ replaced with Cilium

### Version Rules

The components don't all have to run the same version simultaneously. The rules are:

- The `kube-controller-manager`, `kube-scheduler`, and `kubelet` **can't be newer** than the `kube-apiserver`.
- The `kube-controller-manager` and `kube-scheduler` can run up to **one minor version behind** the `kube-apiserver`.
- `kubelet` can run up to **three minor versions behind** the `kube-apiserver`.
- `kubectl` can be one version higher, lower, or the same as the `kube-apiserver`.

> **_Note:_** For simplicity and easier maintenance, it is best to upgrade all components to the same version at once.

---

## The Role of kubeadm in Upgrades

`kubeadm` follows the same versioning as Kubernetes components. The upgrade workflow is:

1. Upgrade `kubeadm` itself to the target version.
2. Run `kubeadm upgrade apply` to upgrade all control plane components and renew certificates.
3. Drain the node to evict its running Pods and mark it unschedulable.
4. Upgrade `kubelet` and `kubectl` manually; `kubeadm` does not manage these.
5. Restart `kubelet`.
6. Uncordon the node for scheduling.

On **worker nodes**, the `kubeadm upgrade node` command only updates the local kubelet configuration because no control plane components are running there. The `drain`, `upgrade`, and `uncordon` steps follow the same pattern.

---

## Understanding Node Draining

Draining a node does two things: it marks the node **unschedulable** and evicts its running Pods. Their controllers then create new Pods on other nodes.

**Drain a node**
```bash
kubectl drain <NODE_NAME> --ignore-daemonsets
```

Kubernetes then notices that the Deployment is missing Pods. Its ReplicaSet creates new ones on another available node.

> **_Warning:_** By default, `kubectl drain` refuses to delete standalone Pods that no controller manages. With `--force`, it deletes them, and nothing recreates them.

Normally, control plane nodes do not run workloads because a `NoSchedule` taint is set on them by default. I removed this taint to run workloads across all nodes.

Two related commands control scheduling:

| Command                   | Effect                                                      |
|---------------------------|-------------------------------------------------------------|
| `kubectl cordon <NODE>`   | Marks the node unschedulable without evicting existing Pods |
| `kubectl uncordon <NODE>` | Marks the node schedulable again                            |

Use `uncordon` after every upgrade to bring the node back into scheduling.

---

## When Should You Upgrade?

Kubernetes supports the **three most recent minor versions**. Each minor version receives about 12 months of regular patches, followed by about 2 months of maintenance mode with only critical fixes. When `v1.35` is released, `v1.32` enters this final window and has roughly two months of support remaining. Plan the upgrade as soon as a new minor version is released.

One important limitation is that **you cannot skip versions**. If you are on `v1.32` and want to reach `v1.35`, you must upgrade through each version in sequence.

```
1.32 → 1.33 → 1.34 → 1.35
```

---

## Before the Upgrade

### Back Up `etcd`

A snapshot of `etcd` lets you restore the cluster state if the upgrade goes wrong. Run it on the control plane node you upgrade, using `etcdctl` and `etcdutl`.

> **_Note:_** kubeadm doesn't install the `etcdctl` and `etcdutl` CLI tools on the node. Install them from the etcd release that matches the etcd version of the cluster.

**Check `etcd` version**
```bash
kubectl -n kube-system exec etcd-k8s-master-1 -- etcdctl version
# etcdctl version: 3.6.6
# API version: 3.6
```

**Create the backup directory**
```bash
sudo mkdir -p /etc/kubernetes/backups
```

**Take an etcd snapshot**
```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /etc/kubernetes/backups/etcd-snapshot-$(date +%Y%m%d).backup
```

> **_Note:_** `etcdctl snapshot status` was deprecated in etcd v3.5 and removed in v3.6. Use `etcdutl snapshot status` instead.

**Check the snapshot**
```bash
sudo etcdutl snapshot status /etc/kubernetes/backups/etcd-snapshot-$(date +%Y%m%d).backup -w table
```

> **_Note:_** Copy the snapshot outside the cluster as well. A backup on the node itself is lost when the node is lost.

### Check the Cilium Version

I replaced kube-proxy with Cilium in my cluster, and each Cilium version supports a range of Kubernetes versions.

**Show the running Cilium version**
```bash
cilium version
```

Compare the running version with the Kubernetes compatibility table in the [Cilium documentation](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/#kubernetes-compatibility). If `v1.35` is not listed, upgrade Cilium first.

---

## Upgrading from v1.34 to v1.35

Upgrade the control plane nodes one at a time, and the worker node last.

The steps of the upgrade:

1. **[Step 1:](#step-1-upgrade-kubeadm-on-the-control-plane-node)** Upgrade kubeadm on the first control plane node.
2. **[Step 2:](#step-2-preview-and-apply-the-control-plane-upgrade)** Preview and apply the control plane upgrade on the node.
3. **[Step 3:](#step-3-check-the-control-plane-pods)** Check the restarted control plane Pods.
4. **[Step 4:](#step-4-drain-the-control-plane-node)** Drain the control plane node.
5. **[Step 5:](#step-5-upgrade-kubelet-and-kubectl)** Upgrade kubelet and kubectl, and restart the kubelet.
6. **[Step 6:](#step-6-uncordon-the-control-plane-node)** Uncordon the control plane node and check the etcd cluster health.
7. **[Step 7:](#step-7-upgrade-the-remaining-control-plane-nodes)** Repeat the steps on the other control plane nodes.
8. **[Step 8:](#step-8-upgrade-the-worker-node)** Upgrade the worker node.

### Step 1: Upgrade kubeadm on the Control Plane Node

SSH into the control plane node and switch to the root user.

**SSH and switch to the root user**
```bash
ssh <USER>@<CONTROL_PLANE_IP>
sudo -i
```
As root, you can skip the `sudo` prefix.

The cluster is installed and configured with `kubeadm` and uses community-owned package repositories.

**Update the apt repository to point to the new Kubernetes version**
```bash
vim /etc/apt/sources.list.d/kubernetes.list
```

**Change the version path in the file to `1.35`**
```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
```

**Update the package index**
```bash
apt update
```

**Find the latest patch release for Kubernetes v1.35**
```bash
apt-cache madison kubeadm
```
This shows a list of available v1.35 patches.
```
 kubeadm | 1.35.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb
 ...
 kubeadm | 1.35.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb
```

**Install the latest version of `kubeadm`**
```bash
apt-mark unhold kubeadm && apt-get update && \
apt-get install kubeadm='1.35.5-1.1' -y && \
apt-mark hold kubeadm
```
- `apt-mark unhold`: Removes the hold on the package. A held package is pinned at its current version and skipped during upgrades.
- `apt-mark hold`: Re-holds the package at the newly installed version, preventing it from being accidentally upgraded by a future `apt-get upgrade`.

**Verify the new version**
```bash
kubeadm version --output short
# v1.35.5
```

### Step 2: Preview and Apply the Control Plane Upgrade

**Before making any changes, check what will be upgraded**
```bash
kubeadm upgrade plan
```

This prints the current cluster version, kubeadm version, all control plane component versions, and the versions they will be upgraded to. It does not make any changes.

```
...
W0610 20:42:43.705266  191300 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" not found
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE           CURRENT   TARGET
kubelet     k8s-master-1   v1.34.8   v1.35.5
kubelet     k8s-master-2   v1.34.8   v1.35.5
kubelet     k8s-master-3   v1.34.8   v1.35.5
kubelet     k8s-worker-1   v1.34.8   v1.35.5

Upgrade to the latest stable version:

COMPONENT                 NODE           CURRENT   TARGET
kube-apiserver            k8s-master-1   v1.34.8   v1.35.5
kube-controller-manager   k8s-master-1   v1.34.8   v1.35.5
kube-scheduler            k8s-master-1   v1.34.8   v1.35.5
kube-proxy                               1.34.8    v1.35.5
CoreDNS                                  v1.12.1   v1.13.1
etcd                      k8s-master-1   3.6.5-0   3.6.6-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.35.5
```

The warning on `kube-proxy` is because kubeadm is unaware that it has been replaced by Cilium in this setup. After the upgrade, check that kube-proxy didn't come back (see Step 3).

The output also shows the exact command to apply the upgrade.

**Upgrade the node**
```bash
kubeadm upgrade apply v1.35.5
```

Confirm when prompted. kubeadm updates all control plane components it manages and renews the cluster certificates. When it completes, it prints a success message:

```
[upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.35.5".
[upgrade] Now please proceed with upgrading the rest of the nodes by following the right order.
```

> **_Note:_** kubeadm regenerates the static Pod manifests from `kubeadm-config`. Flags you added by hand are lost unless they are in the ConfigMap. See the [kube-bench project](/articles/kubernetes/cks/1-kube-bench/#persist-and-roll-out-the-fixes).

### Step 3: Check the Control Plane Pods

After `kubeadm upgrade apply` completes, check the kube-apiserver static Pod with `crictl`, the CLI for CRI container runtimes. `kubectl` still works through HAProxy, but `crictl` shows the container on this node directly, also when it keeps crashing.

**Check the status**
```bash
crictl ps -a | grep kube-apiserver
```

When the static Pod has restarted, you can check the logs using the container ID.

**Show the kube-apiserver logs**
```bash
crictl logs <CONTAINER_ID>
```

With no errors, exit the root user and check the newly running Pods.

**Exit the root user**
```bash
exit
```

**Verify that the control plane Pods were restarted**
```bash
kubectl -n kube-system get pods | grep k8s-master-1
```
```
etcd-k8s-master-1                      1/1     Running   0         10m
kube-apiserver-k8s-master-1            1/1     Running   0         5m
kube-controller-manager-k8s-master-1   1/1     Running   0         9m
kube-scheduler-k8s-master-1            1/1     Running   0         9m
```

They show a recent age. However, `kubectl get nodes` still shows the old version on the control plane node because the `kubelet` hasn't been upgraded yet.

**Check that kube-proxy didn't come back**
```bash
kubectl -n kube-system get ds kube-proxy
```
```
Error from server (NotFound): daemonsets.apps "kube-proxy" not found
```
If the DaemonSet exists, kube-proxy runs next to Cilium. Remove it again.

**Delete the kube-proxy DaemonSet**
```bash
kubectl -n kube-system delete ds kube-proxy
```

**Delete the kube-proxy ConfigMap**
```bash
kubectl -n kube-system delete cm kube-proxy
```

### Step 4: Drain the Control Plane Node

Before upgrading `kubelet`, drain the node. This ensures that no new Pods are scheduled on it while the kubelet is being replaced, and that any running Pod is safely relocated to other nodes.

**Drain the node**
```bash
kubectl drain k8s-master-1 --ignore-daemonsets
```

This cordons the node, which disables scheduling, and evicts all existing Pods. The flag skips DaemonSet Pods because they run on every node and can't move to another node.

**Check that the node is in the scheduling-disabled state**
```bash
kubectl get nodes
```
```
NAME           STATUS                     ROLES           AGE   VERSION
k8s-master-1   Ready,SchedulingDisabled   control-plane   92d   v1.34.8
k8s-master-2   Ready                      control-plane   92d   v1.34.8
k8s-master-3   Ready                      control-plane   92d   v1.34.8
k8s-worker-1   Ready                      <none>          92d   v1.34.8
```

### Step 5: Upgrade kubelet and kubectl

**Switch to the root user**
```bash
sudo -i
```

**Install the new versions**
```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet='1.35.5-1.1' kubectl='1.35.5-1.1' && \
apt-mark hold kubelet kubectl
```

**Reload the systemd daemon and restart kubelet**
```bash
systemctl daemon-reload
systemctl restart kubelet
```

**Verify kubelet is running**
```bash
systemctl status kubelet
```

**Exit the root user**
```bash
exit
```

### Step 6: Uncordon the Control Plane Node

**Uncordon the node**
```bash
kubectl uncordon k8s-master-1
```

**Check the nodes again**
```bash
kubectl get nodes
```

The control plane node should now show the new version and a `Ready` status with no scheduling restriction:

```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master-1   Ready    control-plane   92d   v1.35.5
k8s-master-2   Ready    control-plane   92d   v1.34.8
k8s-master-3   Ready    control-plane   92d   v1.34.8
k8s-worker-1   Ready    <none>          92d   v1.34.8
```

Before moving to the next control plane node, check that all etcd members are healthy. With three members, etcd keeps working when one member is down, but not two.

**Check the etcd cluster health**
```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster
```
```
https://<K8S_MASTER_1_IP>:2379 is healthy: successfully committed proposal: ...
https://<K8S_MASTER_2_IP>:2379 is healthy: successfully committed proposal: ...
https://<K8S_MASTER_3_IP>:2379 is healthy: successfully committed proposal: ... 
```

### Step 7: Upgrade the Remaining Control Plane Nodes

Repeat the process on the other control plane nodes, one at a time. The only difference is the **`kubeadm upgrade node`** command.

> **_Note:_** Only the **first** control plane node runs `kubeadm upgrade apply`. All other control plane nodes use **`kubeadm upgrade node`** instead.

The steps on the other control plane node are the same as Steps 1 to 6 on the first control plane (`k8s-master-1`) node; see [the steps of the upgrade](#upgrading-from-v134-to-v135).

**SSH and switch to root user**
```bash
ssh <USER>@<CONTROL_PLANE_IP>
sudo -i
```

**Update the Kubernetes apt repository to `1.35`**
```bash
vim /etc/apt/sources.list.d/kubernetes.list
```

**Update the package list**
```bash
apt update
```

**Find the latest patch version**
```bash
apt-cache madison kubeadm
```

**Upgrade kubeadm**
```bash
apt-mark unhold kubeadm && apt-get update && \
apt-get install -y kubeadm='1.35.5-1.1' && \
apt-mark hold kubeadm
```

**Verify the new version**
```bash
kubeadm version --output short
```

**Upgrade the node**
```bash
kubeadm upgrade node
```

This upgrades the node. When the upgrade is finished, check the output before you drain the node, upgrade `kubelet` and `kubectl`, and uncordon it, as on `k8s-master-1`.

```
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upgrade/control-plane] The control plane instance for this node was successfully upgraded!
[upgrade/kubeconfig] The kubeconfig files for this node were successfully upgraded!
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
```

**Exit the root user**
```bash
exit
```

**Drain the node**
```bash
kubectl drain k8s-master-2 --ignore-daemonsets
```

**Switch to root user**
```bash
sudo -i
```

**Upgrade kubelet and kubectl**
```bash
apt-mark unhold kubelet kubectl && apt-get update && \
apt-get install -y kubelet='1.35.5-1.1' kubectl='1.35.5-1.1' && \
apt-mark hold kubelet kubectl
```

**Reload and restart kubelet**
```bash
systemctl daemon-reload
systemctl restart kubelet
```

**Exit the root user**
```bash
exit
```

**Uncordon the node**
```bash
kubectl uncordon k8s-master-2
```

**Verify the node**
```bash
kubectl get node k8s-master-2
```

Check the etcd cluster health again, as in [Step 6](#step-6-uncordon-the-control-plane-node) before you update the other control plane node.
#### Repeat for `k8s-master-3`

### Step 8: Upgrade the Worker Node

Upgrade worker nodes one at a time as well, and keep enough capacity to run the workloads.

SSH into the worker node and switch to the root user. Update the `apt` repository the same way as the control plane.

**SSH and switch to the root user**
```bash
ssh <USER>@<WORKER_NODE_IP>
sudo -i
```

**Update the version path to `1.35`**
```bash
vim /etc/apt/sources.list.d/kubernetes.list
# deb [signed-by=...] https://...:/v1.35/deb/ /
```

**Upgrade kubeadm**
```bash
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm='1.35.5-1.1' && \
apt-mark hold kubeadm
```

> **_Note:_** `kubeadm upgrade plan` doesn't work on worker nodes.

**Upgrade the node**
```bash
kubeadm upgrade node
```

On a worker node, `kubeadm upgrade node` does two things:

1. **Downloads the kubelet configuration** from the `kubelet-config` ConfigMap in the cluster.
2. **Writes it to `/var/lib/kubelet/config.yaml`** on the node.

It doesn't install any binaries or restart any services; those steps are manual.

**From a control plane node, drain the worker node**
```bash
kubectl drain k8s-worker-1 --ignore-daemonsets
```
- `--ignore-daemonsets`: Drain the node without evicting the DaemonSet-managed Pods.

If you see an error about Pods not managed by a Deployment, ReplicaSet, StatefulSet, or similar controller, add the `--force` option.

> **_Note:_** In production, avoid standalone Pods without a controller. Nothing recreates them after a drain.

If you see an error about Pods with local storage, add `--delete-emptydir-data`. The data in their `emptyDir` volumes **will be deleted**.

**Drain a node with Pods that use local storage**
```bash
kubectl drain k8s-worker-1 --ignore-daemonsets --delete-emptydir-data
```
- `--delete-emptydir-data`: Accept data loss on `emptyDir` volumes. Because `emptyDir` data is stored on the node, draining the node destroys it.

Drain uses the Eviction API. It respects PodDisruptionBudgets, and a strict PodDisruptionBudget can make the drain wait until enough replicas are available elsewhere.

> **_Note:_** Longhorn can block a drain with its own PodDisruptionBudgets, to protect the last healthy replica of a volume. If the drain hangs, check the Longhorn **Node Drain Policy** setting and the health of the volume replicas.

**Verify that the Pods have relocated**
```bash
kubectl get pods -A -o wide
```

**Back on the worker node, upgrade kubelet and kubectl**
```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y \
  kubelet='1.35.5-1.1' kubectl='1.35.5-1.1' && \
apt-mark hold kubelet kubectl
```

**Reload and restart kubelet**
```bash
systemctl daemon-reload
systemctl restart kubelet
```

**Exit the root user**
```bash
exit
```

**From a control plane node, uncordon the worker node**
```bash
kubectl uncordon k8s-worker-1
```

**Repeat for other worker nodes.**

---

## Final Verification

Once all nodes are upgraded, check the cluster from the control plane:

**List the nodes**
```bash
kubectl get nodes
```
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master-1   Ready    control-plane   92d   v1.35.5
k8s-master-2   Ready    control-plane   92d   v1.35.5
k8s-master-3   Ready    control-plane   92d   v1.35.5
k8s-worker-1   Ready    <none>          92d   v1.35.5
```

All nodes show the new Kubernetes version and a `Ready` status. With three control plane nodes and workloads spread across all nodes, the application remained available throughout the entire upgrade process with no downtime.

---