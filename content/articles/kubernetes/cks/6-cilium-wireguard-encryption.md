---
title: Cilium WireGuard Encryption
date: 2026-11-08
weight: 6
draft: false
reviewed: true
tags:
  - CKS
  - Security
  - Kubernetes
  - Cilium
  - Encryption
  - Tutorial
---
# Project Cilium WireGuard Encryption

In this project, I enabled encryption for Pod traffic in my homelab cluster. Before the change, Pods on different nodes communicated in plain text over the network. Anyone with access to the network could read every HTTP request and response. After the change, Cilium routes that traffic through a WireGuard tunnel between the nodes.

When setting up the cluster with kubeadm, I installed Cilium as the CNI plugin, replacing the default `kube-proxy`. It handles networking with eBPF programs in the Linux kernel, while WireGuard provides built-in kernel-level VPN functionality. Applications don't need to change or even know that their traffic is encrypted.

---

## The Plan

Cilium already runs in the cluster, with `kube-proxy` replacement and native routing between the nodes. I don't install it again. I change its configuration and restart the Cilium agents.

First, I check that Pod traffic is crossing the network in plain text. Then, I enable WireGuard, make sure every node has a tunnel to the others, and capture the same traffic again.

The project uses these names and values:

| Item             | Value                                                                                 |
| ---------------- | ------------------------------------------------------------------------------------- |
| Encryption type  | WireGuard, on UDP port `51871`                                                        |
| WireGuard device | `cilium_wg0` on every node                                                            |
| Server           | Deployment `nginx` with a Service `nginx`, on `k8s-worker-1`                          |
| Client           | Pod `curlpod` on `k8s-master-1`                                                       |
| Capture          | `tcpdump` on `k8s-master-1`, on the network interface of the node and on `cilium_wg0` |

> **_Note:_** Cilium encrypts traffic only between Pods on different nodes. Traffic between two Pods on the same node isn't encrypted. The server and the client must run on different nodes.

---

## What Is eBPF

**extended Berkeley Packet Filter (eBPF)** runs small programs inside the Linux kernel without modifying the kernel source code or using kernel modules. The original BPF only filtered network packets, for example, for `tcpdump`. eBPF can do much more.

An eBPF program goes through these steps:

1. The program is compiled into eBPF bytecode and loaded into the kernel.
2. The kernel verifies the program can't crash it, loop forever, or access unauthorized memory.
3. The kernel compiles bytecode into machine code and attaches it to a hook, such as a network interface or system call.
4. On each event, the program runs and can read, modify, drop, or redirect data, sharing it with user space through eBPF maps.

The program executes within the kernel at the time the event occurs. Packets don't travel to user space and back, and the kernel does not need to evaluate long chains of iptables rules. This keeps eBPF fast.

---

## How Cilium WireGuard Works

Cilium calls this feature transparent encryption. WireGuard is a small and fast VPN protocol, built into the Linux kernel. Cilium uses it to encrypt Pod traffic between the nodes.

After enabling WireGuard, Cilium sets up the tunnels in these steps:

1. The Cilium agent on each node generates a key pair and creates a WireGuard interface device, `cilium_wg0`.
2. Each node publishes its public key in the `network.cilium.io/wg-pub-key` annotation of its CiliumNode object. The private key never leaves the node.
3. Each node sets up a tunnel to every other node on UDP port `51871`.
4. Cilium routes Pod traffic to another node through `cilium_wg0`. WireGuard encrypts the traffic before it leaves the node, and the other node decrypts it.

A Pod still sends and receives plain text. The encryption happens on the node, outside the Pod.

WireGuard does not protect all traffic within the cluster:

- Traffic between Pods on the same node is not encrypted.
- Traffic between the nodes themselves is not encrypted by default.
- WireGuard does not control which Pods can talk to each other. A NetworkPolicy does that.

---

## Step 1: Check the Nodes

WireGuard needs kernel support on every node. Linux v5.6 or newer supports this.

**List the nodes with their kernel versions**
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,KERNEL:.status.nodeInfo.kernelVersion
```
```
NAME           KERNEL
k8s-master-1   6.8.0-147-generic
k8s-master-2   6.8.0-147-generic
k8s-master-3   6.8.0-147-generic
k8s-worker-1   6.8.0-147-generic
```

Every node runs kernel `6.8`, which has WireGuard built in.

The Cilium agent runs as the DaemonSet `cilium` in `kube-system`. The `cilium-dbg` tool shows the agent's state.

**Show the routing and the `kube-proxy` replacement of the Cilium agent**
```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status | grep -E 'KubeProxy|Routing'
```
```
KubeProxyReplacement:    True   [eno1   fe80::9afa:9bff:fea8:5160 192.168.3.130 (Direct Routing)]
Routing:                 Network: Native   Host: Legacy
```

Cilium replaced `kube-proxy` and routes the Pod traffic natively between the nodes, without a VXLAN tunnel.

**Show the encryption status of the Cilium agent**
```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status | grep Encryption
```
```
Encryption:              Disabled
```

With WireGuard, each node publishes its public key in the annotation `network.cilium.io/wg-pub-key` of its CiliumNode object.

**Check the WireGuard public keys of the nodes**
```bash
kubectl get ciliumnodes -o custom-columns=NAME:.metadata.name,WGKEY:.metadata.annotations.network\\.cilium\\.io/wg-pub-key
```
```
NAME           WGKEY
k8s-master-1   <none>
k8s-master-2   <none>
k8s-master-3   <none>
k8s-worker-1   <none>
```

Encryption is disabled, and no node has a WireGuard key yet.

---

## Step 2: Capture the Plain Text Traffic

A capture on the network shows what an attacker on that network can read. I run an nginx server on the worker node and a client on `k8s-master-1`.

**Create the nginx Deployment**
```bash
kubectl create deployment nginx --image=nginx --replicas=1 --port=80
```

> **_Note:_** By default, the control plane nodes have the taint `node-role.kubernetes.io/control-plane:NoSchedule`. The scheduler schedules the nginx Pod on the `k8s-worker-1` node without the taint. In my homelab, I removed the taint to run workloads on all nodes.

**Expose the Deployment with a Service**
```bash
kubectl expose deployment nginx --port=80 --target-port=80
```

The client must run on another node. The `nodeName` skips the scheduler and places the Pod on `k8s-master-1` directly.

**Create the client Pod on `k8s-master-1`**
```bash
kubectl run curlpod --image=rapidfort/curl \
  --labels=app=curlpod \
  --overrides='{"spec":{"nodeName":"k8s-master-1"}}' -- sleep infinity
```
- `--labels`: Gives the Pod a label.
- `--overrides`: Adds `nodeName` to the Pod spec.
- `sleep infinity`: Keeps the container running until you delete the Pod.

**Show the nodes and IP addresses of the Pods**
```bash
kubectl get pods -o wide
```
```
NAME                     READY   STATUS    IP           NODE
curlpod                  1/1     Running   10.0.3.141   k8s-master-1
nginx-75fdcbbc74-pqlvb   1/1     Running   10.0.0.122   k8s-worker-1
```

The nginx Pod runs on `k8s-worker-1`, and the `curlpod` runs on `k8s-master-1`.

**Send a request every two seconds in a second terminal**
```bash
watch "kubectl exec curlpod -- curl -s http://nginx"
```
```
...
<title>Welcome to nginx!</title>
...
```

Keep the command running. `tcpdump` requires the network interface name of the node.

**Find the network interface of `k8s-master-1`**
```bash
ip -br -4 addr
```
```
lo                       UNKNOWN   127.0.0.1/8
eno1                     UP        192.168.3.110/24 metric 100
cilium_host@cilium_net   UP        10.0.3.199/32
```
- `eno1`: The interface with the IP address of the node, which sends the traffic to other nodes.
- `cilium_host`: The internal device of Cilium, which sends the traffic to other Pods on the same node.

**Install the `tcpdump` package**
```bash
sudo apt update && sudo apt install tcpdump
```

**Capture the HTTP traffic on the network interface**
```bash
sudo tcpdump -n -i eno1 -A -c 20 tcp port 80
```
```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
...
10:35:46.962642 IP 10.0.3.141.49236 > 10.0.0.122.80: Flags [P.], seq 1:70, ack 1, win 502, options [nop,nop,TS val 3702765922 ecr 586236384], length 69: HTTP: GET / HTTP/1.1
E..y..@.>.8.
...
..z.T.P.1o&...	.....r.....
...b".A.GET / HTTP/1.1
Host: nginx
User-Agent: curl/8.21.0
Accept: */*
...
10:35:46.963274 IP 10.0.0.122.80 > 10.0.3.141.49236: Flags [P.], seq 1:239, ack 70, win 509, options [nop,nop,TS val 586236385 ecr 3702765922], length 238: HTTP: HTTP/1.1 200 OK
E.."..@.>.0;
..z
....P.T...	.1ok...........
".A....bHTTP/1.1 200 OK
Server: nginx/1.31.8
Date: Mon, 09 Nov 2026 09:35:46 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Wed, 21 Oct 2026 14:24:51 GMT
Connection: keep-alive
ETag: "6ad8cb33-380"
Accept-Ranges: bytes
...
10:35:46.963429 IP 10.0.0.122.80 > 10.0.3.141.49236: Flags [P.], seq 239:1135, ack 70, win 509, options [nop,nop,TS val 586236385 ecr 3702765922], length 896: HTTP
E.....@.>.-.
..z
....P.T.....1ok....o......
".A....b<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
20 packets captured
24 packets received by filter
0 packets dropped by kernel
```
- `-n`: Shows IP addresses, without DNS lookups.
- `-i`: The network interface to capture on.
- `-A`: Shows the content of each packet as text.
- `-c 20`: Stops after 20 packets.
- `tcp port 80`: Only HTTP traffic.

This shows the `GET / HTTP/1.1` request and the nginx welcome page in plain text. The packets travel directly from the IP address of `curlpod` to the IP address of the nginx Pod. Cilium translates the Service address before the packet leaves the Pod.

---

## Step 3: Enable WireGuard

To enable encryption across the entire cluster, configure it in the Cilium configuration. The steps to change the configuration depend on how you installed Cilium. In my homelab, I installed Cilium with the Cilium CLI when I set up the kubeadm cluster.

You don't need to upgrade or change the nodes. The kernel already supports WireGuard, and the Cilium agent on each node generates its own keys and creates a WireGuard device after the restart.

**Check if WireGuard is enabled**
```bash
kubectl -n kube-system get configmap cilium-config -o yaml | grep enable-wireguard:
```

No output means that WireGuard is disabled.

The Cilium CLI manages Cilium as a Helm release named `cilium`. Helm values are the source of truth, not the ConfigMap. The next `cilium upgrade` can undo manual changes to the ConfigMap. Use the Cilium CLI to change the values.

### With Cilium CLI

`cilium upgrade` must use the version that already runs. `--reuse-values` keeps the values of the current release. With a new version, it ignores the chart's new values.

**Check the version**
```bash
cilium version
```
```
cilium-cli: v0.19.2 compiled with go1.25.5 on linux/amd64
cilium image (default): v1.19.1
cilium image (stable): v1.20.4
cilium image (running): 1.19.1
```

The cluster runs Cilium `1.19.1`.

**Enable WireGuard with the Cilium CLI**
```bash
cilium upgrade --version 1.19.1 \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
```
- `--version 1.19.1`: The running version, from `cilium version`.
- `--reuse-values`: Keeps the values of the current release.
- `encryption.enabled`: Turns on network encryption.
- `encryption.type`: The encryption type, `wireguard` or `ipsec`.

### Option with the ConfigMap

The Cilium agents read their settings from the ConfigMap `cilium-config`. Use this method only if you installed Cilium without the Cilium CLI or Helm.

**Enable WireGuard in the Cilium configuration**
```bash
kubectl -n kube-system patch configmap cilium-config --type merge -p '{"data":{"enable-wireguard":"true"}}'
```

### Restart the Cilium Agents

The agents only read the configuration at startup. Restart them to load the new setting.

**Restart the Cilium agents**
```bash
kubectl -n kube-system rollout restart daemonset/cilium
```

**Wait until all agents are back**
```bash
kubectl -n kube-system rollout status daemonset/cilium
```
```
Waiting for daemon set "cilium" rollout to finish: 2 out of 4 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 2 out of 4 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 2 out of 4 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 2 out of 4 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 3 out of 4 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 2 of 4 updated pods are available...
Waiting for daemon set "cilium" rollout to finish: 3 of 4 updated pods are available...
daemon set "cilium" successfully rolled out
```

**Check that WireGuard is enabled**
```bash
kubectl -n kube-system get configmap cilium-config -o yaml | grep enable-wireguard:
```
```
  enable-wireguard: "true"
```

---

## Step 4: Verify the Tunnels

After the restart, each agent creates a key pair and sets up a WireGuard tunnel to every other node.

**Show the encryption status of the Cilium agent**
```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status | grep Encryption
```
```
Encryption:              Wireguard       [NodeEncryption: Disabled, cilium_wg0 (Pubkey: 7z6F2ZuVf5IS/sQHssXZ191P2kj/HXcTQnff+hQjlF4=, Port: 51871, Peers: 3)]
```
- `Wireguard`: WireGuard encryption is active.
- `NodeEncryption: Disabled`: Cilium only encrypts traffic between Pods. Traffic between the nodes themselves stays plain text unless you set `encryption.nodeEncryption=true`.
- `cilium_wg0`: The WireGuard interface device on the node.
- `Port: 51871`: The UDP port of the tunnel.
- `Peers`: The number of nodes with a tunnel, minus the node of the Cilium agent that answers. With four nodes, it's `3`.

**List the WireGuard public keys of the nodes**
```bash
kubectl get ciliumnodes -o custom-columns=NAME:.metadata.name,WGKEY:.metadata.annotations.network\\.cilium\\.io/wg-pub-key
```
```
NAME           WGKEY
k8s-master-1   7z6F2ZuVf5IS/sQHssXZ191P2kj/HXcTQnff+hQjlF4=
k8s-master-2   PjZ6gbCHsjq7DZCfffN5eGWYmrB7ipa5kzMrTu0fDA8=
k8s-master-3   KVvqjU9FcOCn0O8kmYubg6t3dPvyBBCBK0VlLK/UrX0=
k8s-worker-1   vgsndkk2VmNjGe6vTBICdWzdn5C1zDmB2TvSAXNJz0Y=
```

Every node now has a public key. The private key never leaves the node. Cilium shares only the public keys, through the CiliumNode objects.

---

## Step 5: Capture the Traffic Again

I repeat the capture from step 2, where the `watch` command sends a request every two seconds.

**Capture the HTTP traffic on the network interface**
```bash
sudo tcpdump -n -i eno1 -A -c 20 tcp port 80
```
```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
3 packets received by filter
0 packets dropped by kernel
```

The capture stays empty. Stop it with `Ctrl+C`. The HTTP traffic no longer crosses the network as TCP on port `80`.

**Capture the WireGuard traffic on the network interface**
```bash
sudo tcpdump -n -i eno1 -c 20 udp port 51871
```
```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
13:21:15.253041 IP 192.168.3.110.51871 > 192.168.3.140.51871: UDP, length 112
13:21:15.254139 IP 192.168.3.140.51871 > 192.168.3.110.51871: UDP, length 112
13:21:15.254480 IP 192.168.3.110.51871 > 192.168.3.140.51871: UDP, length 96
13:21:15.377591 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 96
13:21:15.378153 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 96
13:21:15.378506 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 96
13:21:15.378643 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 112
13:21:15.378712 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 96
13:21:15.379160 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 96
13:21:15.379160 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 96
13:21:15.379340 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 112
13:21:15.379530 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 96
13:21:15.379632 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 96
13:21:15.379876 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 96
13:21:15.379902 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 256
13:21:15.380792 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 96
13:21:15.380792 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 128
13:21:15.380793 IP 192.168.3.130.51871 > 192.168.3.110.51871: UDP, length 608
13:21:15.381103 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 96
13:21:15.381455 IP 192.168.3.110.51871 > 192.168.3.130.51871: UDP, length 144
20 packets captured
156 packets received by filter
0 packets dropped by kernel
```

The same traffic now appears as UDP packets between the nodes' IP addresses on port `51871`. The content is encrypted. An attacker on the network sees which nodes communicate with each other but not what the Pods send.

The Cilium agent runs in the network namespace of the node. The device `cilium_wg0` is visible on the node itself.

**Capture the traffic on the WireGuard device**
```bash
sudo tcpdump -n -i cilium_wg0 -A -c 20
```
```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on cilium_wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
...
13:21:23.237930 IP 10.0.3.123.33619 > 10.0.1.131.53: 9283+ AAAA? nginx.labs.svc.cluster.local. (46)
E..J..@.?...
..{
....S.5.6.E$C...........nginx.labs.svc.cluster.local.....
...
13:21:23.240227 IP 10.0.3.123.37930 > 10.0.0.116.80: Flags [P.], seq 1:70, ack 1, win 507, options [nop,nop,TS val 1431961092 ecr 59436558], length 69: HTTP: GET / HTTP/1.1
E..y.$@.?..m
..{
..t.*.PQeI..........Z.....
UY......GET / HTTP/1.1
Host: nginx
User-Agent: curl/8.21.0
Accept: */*

...
13:21:23.241359 IP 10.0.0.116.80 > 10.0.3.123.37930: Flags [P.], seq 1:239, ack 70, win 502, options [nop,nop,TS val 59436559 ecr 1431961092], length 238: HTTP: HTTP/1.1 200 OK
E..".
@.?...
..t
..{.P.*....QeI.....N......
....UY..HTTP/1.1 200 OK
Server: nginx/1.31.8
Date: Mon, 09 Nov 2026 12:21:23 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Wed, 21 Oct 2026 14:24:51 GMT
Connection: keep-alive
ETag: "6ad8cb33-380"
Accept-Ranges: bytes


20 packets captured
29 packets received by filter
0 packets dropped by kernel
```

On `cilium_wg0`, the request from `curlpod` to nginx is still readable: the DNS lookup for `nginx.labs.svc.cluster.local`, the `GET / HTTP/1.1` request, and the `HTTP/1.1 200 OK` response, with the Pod IP addresses. `tcpdump` captures the packets before WireGuard encrypts them. This capture proves that Pod traffic goes through the tunnel. The capture on the network interface proves the traffic is encrypted in transit.

### Compare Both Sides of the Tunnel

The last check compares the same traffic on both sides of the tunnel, during the same 10 seconds. `grep` counts the lines with readable HTTP text.

**Count the readable HTTP in the encrypted traffic on the network**
```bash
sudo timeout 10 tcpdump -n -l -i eno1 -A udp port 51871 | grep -c -E 'GET /|nginx'
```
```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
2450 packets captured
2453 packets received by filter
0 packets dropped by kernel
0
```
- `timeout 10`: Stops `tcpdump` after 10 seconds.
- `-l`: Sends each line to `grep` right away, instead of in blocks.
- `grep -c -E 'GET /|nginx'`: Counts the lines with an HTTP request or the name `nginx`.

**Count the readable HTTP inside the tunnel**
```bash
sudo timeout 10 tcpdump -n -l -i cilium_wg0 -A tcp port 80 | grep -c -E 'GET /|nginx'
```
```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on cilium_wg0, link-type RAW (Raw IP), snapshot length 262144 bytes
60 packets captured
60 packets received by filter
0 packets dropped by kernel
55
```

On the network, `grep` finds `0` readable lines in 2450 packets: the requests cross the network, but nothing in them is readable. Inside the tunnel, `grep` finds `55` lines in 60 packets: the same requests are readable before WireGuard encrypts them. The same traffic is readable on one side of the tunnel and not on the other.

> **_Note:_** Capturing on `eth0` inside a Pod doesn't prove anything. A Pod always sends and receives plain text. The encryption happens on the node, after the packet leaves the Pod.

---

## Cleanup

Stop the `watch` command with `Ctrl+C`. Keep WireGuard enabled: it protects all Pod traffic between the nodes from now on.

**Delete the client Pod**
```bash
kubectl delete pod curlpod
```

**Delete the Service**
```bash
kubectl delete service nginx
```

**Delete the Deployment**
```bash
kubectl delete deployment nginx
```

---

## Common Errors

| Error                                                    | Cause                                                                                    | Fix                                                                    |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `Encryption: Disabled` after the restart                 | The agents didn't restart, or `cilium upgrade` undid the ConfigMap                       | Check the configuration, and restart the DaemonSet                     |
| `Peers` is lower than the number of nodes minus one      | A node can't reach the others on UDP port `51871`                                        | Open UDP port `51871` between all nodes                                |
| The capture on `cilium_wg0` stays empty                  | Both Pods run on the same node                                                           | Check the nodes with `kubectl get pods -o wide`, and use `nodeName`    |
| `tcpdump: cilium_host@cilium_net: No such device exists` | The `@cilium_net` suffix isn't part of the name, and `cilium_host` is an internal device | Capture on the interface with the node IP, such as `eno1`              |
| Pods lose their connections during the change            | The agents restart and set up the tunnels                                                | Wait for `rollout status`, and plan the change in a maintenance window |

---

## Conclusion

WireGuard protects Pod traffic on the network between nodes. It doesn't change what a Pod can access. Any Pod can still communicate with any other Pod, encrypted or not. A NetworkPolicy or CiliumNetworkPolicy determines which Pods can communicate with each other. Use both for better security.

Compared to a service mesh such as Istio, WireGuard is simpler: nodes create and share keys, and Pods don't need sidecars. Encryption happens at the node level rather than per workload, and WireGuard doesn't verify each Pod's identity as mTLS does.

---

## Reference

| Concept                        | What it means                                                          |
| ------------------------------ | ---------------------------------------------------------------------- |
| Transparent encryption         | Cilium encrypts Pod traffic between nodes, without changes to the apps |
| `enable-wireguard`             | The setting in the ConfigMap `cilium-config` that turns on WireGuard   |
| `encryption.enabled`           | The Helm value that turns on transparent encryption                    |
| `encryption.type`              | The Helm value for the encryption type, `wireguard` or `ipsec`         |
| `cilium_wg0`                   | The WireGuard device on every node                                     |
| UDP port `51871`               | The port of the WireGuard tunnels between the nodes                    |
| `network.cilium.io/wg-pub-key` | The annotation on the CiliumNode object with the public key of a node  |
| `Peers`                        | The number of tunnels to other nodes: the number of nodes minus one    |

- [Cilium docs: Transparent Encryption](https://docs.cilium.io/en/stable/security/network/encryption/)
- [Cilium docs: WireGuard Transparent Encryption](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)

---

## Best Practices

- Check the kernel version and open UDP port `51871` between the nodes before you enable WireGuard.
- Change the configuration the same way you installed Cilium: the Cilium CLI or Helm for a managed release, the ConfigMap otherwise.
- Enable WireGuard in a maintenance window. Connections can drop while the agents restart.
- Check the status after the change. `Peers` must be the number of nodes minus one.
- Test with Pods on different nodes, and capture on the network interface of the node.
- Combine encryption with network policies that limit which Pods can talk to each other.

---

## CKS Exam Takeaways

- Cilium encrypts Pod traffic between nodes with WireGuard or IPsec. Encryption applies to the whole cluster.
- `enable-wireguard: "true"` in `cilium-config`, or the Helm values `encryption.enabled=true` and `encryption.type=wireguard`, enables WireGuard.
- Restart the agents after a ConfigMap change: `kubectl -n kube-system rollout restart daemonset/cilium`.
- `cilium-dbg status | grep Encryption` in the agent shows `Wireguard`, `cilium_wg0`, and the number of peers.
- Traffic between Pods on the same node isn't encrypted.
- `tcpdump` on `cilium_wg0` shows the traffic before encryption. `tcpdump` on the node interface with `udp port 51871` shows the encrypted traffic.
- Encryption doesn't control traffic. Use a NetworkPolicy for that.

---
