---
title: AppArmor for nginx
date: 2026-10-31
weight: 4
draft: false
reviewed: true
tags:
  - CKS
  - Security
  - Kubernetes
  - AppArmor
  - Linux
  - Tutorial
---
# Project AppArmor for nginx





In this project, I built a least-privilege AppArmor profile for an nginx web server in my homelab cluster. It's based on the [RBAC Users](/articles/kubernetes/cks/2-rbac-users/) project, where the `hans-adm` user prepares the cluster, and the `hans-dev` user deploys nginx in the `dev` Namespace.

RBAC decides who can create a Pod and who can run `kubectl exec` in it. It can't control what a process inside the container does. The default profile `cri-containerd.apparmor.d` protects the node but allows almost everything inside the container. If an attacker finds a bug in the web application, they can start a shell, modify the website, or read the ServiceAccount token. A custom AppArmor profile allows only what nginx needs to serve the website. Everything else is blocked and logged.

This is an advanced project that goes beyond the CKS exam. Creating a profile for a real-world application requires several rounds of testing. I used AI to learn how to use `aa-logprof` to add rules and to understand what they do.

---

## The Plan

I don't write the profile from memory. I first run nginx in `complain` mode, read the kernel log to see what it does, and then turn that into rules. It's one profile, in one file, from start to end. I run the AppArmor commands as root on the nodes. I run the `kubectl` commands on my laptop, with the kubeconfig files from the RBAC project.

The profile allows the actions nginx needs and blocks the actions an attacker needs.

| Action                                        | Allowed | Why                                                   |
| --------------------------------------------- | ------- | ----------------------------------------------------- |
| Read `/etc/nginx` and `/usr/share/nginx/html` | Yes     | The configuration and the website                     |
| Write to `/var/cache/nginx` and the PID file  | Yes     | nginx stores temporary files and its process ID there |
| Bind port `80`                                | Yes     | The web server listens on it                          |
| Receive signals from `runc`                   | Yes     | Without it, the Pod can't be stopped                  |
| Start a shell from nginx                      | No      | An attacker who breaks into nginx needs a shell       |
| Run programs such as `ls` from a shell        | No      | An attacker needs programs to look around             |
| Write to `/usr/share/nginx/html`              | No      | Stops an attacker from changing the website           |
| Read the ServiceAccount token                 | No      | Stops an attacker from calling the Kubernetes API     |
|                                               |         |                                                       |

Two things about the official `nginx` image matter for this project:

- It starts with a shell script, `/docker-entrypoint.sh`, before starting nginx. The profile blocks the programs that this script needs. That's why the Pods in this project start `nginx` directly, without the script.
- It runs as root. nginx needs root privileges to prepare its cache directories and to start its worker processes as user `101`. That's why the profile allows `chown`, `setuid`, and `setgid`. That's also why the Namespace `dev` uses `baseline`, not `restricted`, because `restricted` doesn't allow containers that run as root.

> **_Note:_** If you don't want root, use the [`nginxinc/nginx-unprivileged`](https://hub.docker.com/r/nginx/nginx-unprivileged/) image. It runs as a normal user, on port `8080`, and works with `restricted`.

Each user does the part that their permissions allow:

| Part                           | Who               | Why                                                  |
| ------------------------------ | ----------------- | ---------------------------------------------------- |
| Build and load the profile     | root on the nodes | AppArmor runs on the node, outside Kubernetes        |
| Label the nodes                | `hans-adm`        | `edit` can't change Nodes                            |
| Enforce Pod Security on `dev`  | `hans-adm`        | `edit` can't change the Namespace                    |
| Run the Deployment and test it | `hans-dev`        | `edit` can create Deployments and use `kubectl exec` |

The project uses these names and values:

| Item         | Value                                                                   |
| ------------ | ----------------------------------------------------------------------- |
| Profile name | `nginx-least-privilege`                                                 |
| Profile file | `/etc/apparmor.d/nginx-least-privilege` on every node                   |
| Node label   | `apparmor-nginx=loaded`                                                 |
| Pod Security | `baseline` enforced on the Namespace `dev`                              |
| Deployment   | `nginx-server` in the Namespace `dev`, 2 replicas, image `nginx:1.31.8` |

---

## The Tasks

The project takes 7 steps:

1. [Check AppArmor on the nodes](#step-1-check-apparmor-on-the-nodes): Check that AppArmor is turned on, on every node.
    - Docs: [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
2. [Build the profile in complain mode](#step-2-build-the-profile-in-complain-mode): Write a first profile that only logs, and run nginx with it on one node.
    - Docs: [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
3. [Generate the enforce profile](#step-3-generate-the-enforce-profile): Turn the log into rules, block everything else, and fix the profile until nginx starts.
    - Docs: [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
4. [Load the profile on all nodes](#step-4-load-the-profile-on-all-nodes): Copy the profile to every node, and label the nodes.
    - Docs: [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/) and [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
5. [Enforce Pod Security on the Namespace](#step-5-enforce-pod-security-on-the-namespace): Stop users from turning AppArmor off.
    - Docs: [Enforce Pod Security Standards with Namespace Labels](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/) and [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
6. [Run nginx as a Deployment](#step-6-run-nginx-as-a-deployment): Run two nginx replicas with the profile.
    - Docs: [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/) and [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
7. [Test the profile as an attacker](#step-7-test-the-profile-as-an-attacker): Try to start a shell and change the website, and read the blocked actions in the log.
    - Docs: [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)

The [Updating the Profile](#updating-the-profile) and [Remove the Profile](#remove-the-profile) sections cover the maintenance.

> **_Warning:_** A Pod with a `Localhost` profile only starts on a node where the profile is loaded. Load the profile on a new node before you add the label.

---

## Prerequisites

| Item       | Value                                                                                   |
| ---------- | --------------------------------------------------------------------------------------- |
| Cluster    | kubeadm v1.35, containerd, 3 control plane nodes, 1 worker node                         |
| Scheduling | No `NoSchedule` taint on the control plane nodes. Pods run on all four nodes.           |
| Users      | `hans-adm` and `hans-dev` from [RBAC Users](/articles/kubernetes/cks/2-rbac-users/)     |
| Kubeconfig | `~/.kube/rbac-users/hans-adm.conf` and `~/.kube/rbac-users/hans-dev.conf` on the laptop |
| Namespace  | `dev`                                                                                   |
| Access     | SSH with the laptop into every node, with `sudo` rights                                 |
| Tools      | `apparmor_parser`, `journalctl`, and `aa-status` on all nodes                           |
| Packages   | `apparmor-utils` and `rsyslog` on `k8s-worker-1`, for `aa-logprof`                      |

**Install the profile tools on the `k8s-worker-1` node**
```bash
sudo apt update && sudo apt install apparmor-utils rsyslog -y
```
- `apparmor-utils`: Contains `aa-logprof`, `aa-complain`, and `aa-enforce`.
- `rsyslog`: Writes the kernel log to `/var/log/syslog`, where `aa-logprof` reads it.

> **_Note:_** Install `rsyslog` before step 2. It only writes the log entries from after the install.

On the laptop, step 2 sets `KUBECONFIG` to `~/.kube/rbac-users/hans-dev.conf`, as in the RBAC project. Commands for `hans-adm` use `--kubeconfig ~/.kube/rbac-users/hans-adm.conf`.

---

## Step 1: Check AppArmor on the Nodes

A Pod with an AppArmor profile only starts on a node where the AppArmor kernel module is loaded. The kubelet checks this before it starts the container.

**Check the kernel module on every node**
```bash
for node in master-1 master-2 master-3 worker-1; do
  echo -n "$node: "
  ssh $node@k8s-$node cat /sys/module/apparmor/parameters/enabled
done
```
```
master-1: Y
master-2: Y
master-3: Y
worker-1: Y
```
- `Y`: The module is loaded.

Containers without a profile run with the default profile of containerd, `cri-containerd.apparmor.d`. It allows file access, network access, and capabilities. It blocks mounts and writes to most files in `/proc` and `/sys`. That protects the node, but not the application in the container.

**Show the default profile on the `k8s-worker-1` node**
```bash
sudo aa-status | grep cri-containerd
```
```
cri-containerd.apparmor.d
/csi-snapshotter (1013) cri-containerd.apparmor.d
...
/usr/local/bin/helm-controller (4192013) cri-containerd.apparmor.d
```

The first line is the profile. The other lines are container processes that run with it.

---

## Step 2: Build the Profile in Complain Mode

A profile that blocks too many actions prevents nginx from starting. In `complain` mode, AppArmor allows all actions, logs those the profile doesn't allow, and shows what nginx needs. I create the profile on the worker node and copy it to the other nodes in step 4.

### Write the First Profile

The first profile allows almost nothing. Everything nginx does ends up in the log. I build it step by step, in the `/etc/apparmor.d` directory. Each snippet shows the whole file so far.

**Create the profile file on the `k8s-worker-1` node**
```bash
sudo vim /etc/apparmor.d/nginx-least-privilege
```

**Profile block**

Start with the ABI version and a profile block with the profile name:
```
abi <abi/3.0>,

profile nginx-least-privilege {
}
```
- `abi <abi/3.0>`: The AppArmor version of the profile.
- `profile nginx-least-privilege`: The profile name. AppArmor only uses the profile when something asks for it by name, such as Kubernetes with `localhostProfile`.

**Tunables**

To make profiles portable, common paths are defined as variables in `tunables`. You can create your own, but the defaults are usually enough:
```
abi <abi/3.0>,
include <tunables/global>

profile nginx-least-privilege {
}
```
- `include <tunables/global>`: Loads the default variables, such as `@{HOME}`. The abstractions use them.

**Abstractions**

While tunables let you reuse paths, `abstractions` let you reuse rules. They reduce duplication and make profiles easier to maintain:
```
abi <abi/3.0>,
include <tunables/global>

profile nginx-least-privilege {
  include <abstractions/base>
}
```
- `include <abstractions/base>`: Rules that almost every Linux program needs, such as loading shared libraries.

**Flags**

Flags change how the whole profile works:
```
abi <abi/3.0>,
include <tunables/global>

profile nginx-least-privilege flags=(attach_disconnected, complain) {
  include <abstractions/base>
}
```
- `attach_disconnected`: Lets the profile work on paths outside the mount namespace of the container.
- `complain`: Logs the actions the profile doesn't allow, instead of blocking them.

**Signals**

A profile without signal rules blocks all signals. Without a rule for `runc`, the container can't be stopped, and the Pod stays in `Terminating` status:
```
abi <abi/3.0>,
include <tunables/global>

profile nginx-least-privilege flags=(attach_disconnected, complain) {
  include <abstractions/base>

  signal (receive) peer=runc,
}
```
- `signal (receive) peer=runc`: Lets `runc` stop the container when the Pod is deleted.

**Save the file**
```
:wq
```

**Load the profile**
```bash
sudo apparmor_parser -r /etc/apparmor.d/nginx-least-privilege
```
- `-r`: Replaces or loads the profile whether it exists or not.

**Check the mode of the profile**
```bash
sudo grep nginx-least-privilege /sys/kernel/security/apparmor/profiles
# nginx-least-privilege (complain)
```
- `(complain)`: The profile is loaded in `complain` mode.

### Run a Test Pod

User `hans-dev` runs a test Pod with the profile on `k8s-worker-1`. The Pod starts `nginx` directly, without the entrypoint script, as the Deployment does later.

**Use the `hans-dev` kubeconfig as a standard user**
```bash
export KUBECONFIG=~/.kube/rbac-users/hans-dev.conf
```

**Verify the correct user**
```bash
kubectl auth whoami
```
```
ATTRIBUTE                                           VALUE
Username                                            hans-dev
Groups                                              [homelab-devs system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=...a308]
```

This manifest pins the test Pod to the `k8s-worker-1` node, where the profile is loaded:
```bash
cat <<EOF > nginx-profile-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-profile-test
  namespace: dev
spec:
  nodeName: k8s-worker-1
  automountServiceAccountToken: false
  containers:
    - name: nginx
      image: nginx:1.31.8
      command: ["nginx", "-g", "daemon off;"]
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: nginx-least-privilege
EOF
```
- `nodeName`: Runs the Pod on the node with the profile, without the scheduler.
- `automountServiceAccountToken: false`: Doesn't mount the ServiceAccount token. nginx doesn't call the Kubernetes API.
- `image: nginx:1.31.8`: A fixed tag. The profile is built for this image. A new image can need new rules.
- `command`: Starts `nginx` directly. `daemon off;` keeps it in the foreground, as the main process of the container.
- `appArmorProfile`: Runs the container with the profile `nginx-least-privilege`.

**Create the test Pod as `hans-dev` user**
```bash
kubectl -n dev apply -f nginx-profile-test.yaml
```

**Check that the Pod runs**
```bash
kubectl -n dev get pod nginx-profile-test
```
```
NAME                 READY   STATUS    RESTARTS   AGE
nginx-profile-test   1/1     Running   0          7s
```

### Send Requests

nginx opens some files only when it handles a request. Request an existing page and a missing page. These actions then show up in the log too.

**Forward a local port to the Pod**
```bash
kubectl -n dev port-forward pod/nginx-profile-test 8080:80 &
```
- `&`: Runs the port forward in the background.

Open a second terminal on the machine where the port forward runs, and send requests to the nginx server.

**Request the home page**
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/
# 200
```
- `-w "%{http_code}\n"`: Prints only the HTTP status code.

**Request a missing page**
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/missing
# 404
```

**Stop the port forward**
```bash
kill %1
```
- `kill %1`: Stops the first background job, which is the port forward. Run it in the first terminal.

**Delete the test Pod**
```bash
kubectl -n dev delete pod nginx-profile-test
```

The delete action also logs the shutdown of nginx, such as removing `/run/nginx.pid`.

### Collect the Log Entries

AppArmor writes each action that the profile doesn't allow to the kernel log, with `apparmor="ALLOWED"` and the profile name.

**List the actions of the profile on the `k8s-worker-1` node**
```bash
sudo journalctl -k --since "10 min ago" \
  | grep 'apparmor="ALLOWED"' \
  | grep 'profile="nginx-least-privilege"' \
  | grep -oE 'operation=.*'
```
```
operation="mknod" class="file" profile="nginx-least-privilege" name="/var/log/nginx/error.log" pid=560649 comm="nginx" requested_mask="c" denied_mask="c" fsuid=0 ouid=0
operation="mknod" class="file" profile="nginx-least-privilege" name="/proc/1/fd/2" pid=560649 comm="nginx" requested_mask="c" denied_mask="c" fsuid=0 ouid=0
operation="capable" class="cap" profile="nginx-least-privilege" pid=560649 comm="nginx" capability=1  capname="dac_override"
operation="open" class="file" profile="nginx-least-privilege" name="/etc/nginx/nginx.conf" pid=560649 comm="nginx" requested_mask="r" denied_mask="r" fsuid=0 ouid=0
...
operation="accept" class="net" profile="nginx-least-privilege" pid=560662 comm="nginx" lport=80 src=80 family="inet" sock_type="stream" protocol=6 requested="accept" denied="accept"
...
operation="open" class="file" profile="nginx-least-privilege" name="/usr/share/nginx/html/index.html" pid=560662 comm="nginx" requested_mask="r" denied_mask="r" fsuid=101 ouid=0
...
operation="accept" class="net" profile="nginx-least-privilege" pid=560663 comm="nginx" lport=80 src=80 family="inet" sock_type="stream" protocol=6 requested="accept" denied="accept"
...
operation="unlink" class="file" profile="nginx-least-privilege" name="/run/nginx.pid" pid=560649 comm="nginx" requested_mask="d" denied_mask="d" fsuid=0 ouid=0
```
- `grep -oE 'operation=.*'`: Drops the timestamp and the audit header.
- `...`: Shortened. The left-out lines are more files in `/etc` and the network actions of each request.

Each line has an `operation`, such as `open`, `capable`, `mknod`, or `accept`. File entries show the path in `name` and the access in `requested_mask`. Capability entries show the capability in `capname`. In `complain` mode, `denied` shows what the profile would block in `enforce` mode.

`pid=560649` with `fsuid=0` is the master process, which runs as root. The other `pid` values, `560662` and `560663`, are the worker processes. `fsuid=101` shows that they run as the `nginx` user.

In the `nginx` image, `/var/log/nginx/error.log` links to the container's error output. That's why `/proc/1/fd/2` shows up too.

---

## Step 3: Generate the Enforce Profile

I don't write the allow rules by hand. `aa-logprof` is an interactive AppArmor utility used to scan system logs for security events and update the AppArmor profile. It reads the `ALLOWED` entries from step 2 and asks you, for each one, what is allowed, and adds the rules to the same profile file. The profile gets no broad `file,` or `capability,` rule. Every action that isn't in the profile is blocked.

### Generate the Rules

`aa-logprof` reads `/var/log/syslog` and shows the entries for the profiles in `/etc/apparmor.d`. It asks one question per log entry.

**Generate the rules from the log on the `k8s-worker-1` node**
```bash
sudo aa-logprof
```
```
Updating AppArmor profiles in /etc/apparmor.d.
Reading log entries from /var/log/syslog.
Complain-mode changes:

Profile:    nginx-least-privilege
Capability: dac_override
Severity:   9

 [1 - capability dac_override,]
(A)llow / [(D)eny] / (I)gnore / Audi(t) / Abo(r)t / (F)inish
...
```

Each question shows the profile, path, or capability, and access. The option in `[...]` is the selected rule. Press a number to choose another option. Enter alone picks the default, `[(D)eny]`.

| Question                                                                | Keys                                           | Rule                                |
| ----------------------------------------------------------------------- | ---------------------------------------------- | ----------------------------------- |
| `Capability: dac_override`                                              | `A`                                            | `capability dac_override,`          |
| `/var/log/nginx/error.log`                                              | `A`                                            | `owner /var/log/nginx/error.log w,` |
| `/proc/1/fd/2`                                                          | `2`, then `A`                                  | `owner /proc/1/fd/2 w,`             |
| `/etc/nginx/nginx.conf`, `mime.types`, `conf.d/`, `conf.d/default.conf` | `A` for each file                              | `owner /etc/nginx/<FILE> r,`        |
| `/etc/nsswitch.conf`, `/etc/passwd`, `/etc/group`                       | `A`                                            | `owner /etc/<FILE> r,`              |
| `/usr/share/nginx/html/index.html`                                      | `N`, type `/usr/share/nginx/html/**`, then `A` | `/usr/share/nginx/html/** r,`       |
| `/run/nginx.pid`                                                        | `A`                                            | `owner /run/nginx.pid w,`           |
| Save the changes                                                        | `S`                                            |                                     |
- `owner`: The rule only applies when the process owns the file. The master process runs as root, and reads and writes files of root.
- `/etc/nginx/<FILE>`: One rule per configuration file. nginx can only read the files it read in step 2. A new file in `conf.d` needs a new rule.
- `/usr/share/nginx/html/**` without `owner`: The worker processes read the website as the `nginx` user, but the files belong to root.
- `/proc/1/fd/2`: In the container, nginx is process 1. `/var/log/nginx/error.log` is a link to its error output.
- `(N)ew`: Type only the path. `aa-logprof` adds the access by itself. If it asks "Do you really want to use this path?", the path doesn't match the log entry: answer `N`, and type it again.

> **_Warning:_** Don't use `(G)lob` to widen a path. A second `G` can turn `/etc/nginx/*` into `/etc/**`. That rule lets root read every file in `/etc`, `/etc/shadow` included.

**Show the generated profile**
```bash
sudo cat /etc/apparmor.d/nginx-least-privilege
```
```
# Last Modified: Tue Nov  3 11:21:08 2026
abi <abi/3.0>,

include <tunables/global>

profile nginx-least-privilege flags=(attach_disconnected, complain) {
  include <abstractions/base>

  capability dac_override,

  signal receive peer=runc,

  /usr/share/nginx/html/** r,
  owner /etc/group r,
  owner /etc/nginx/conf.d/ r,
  owner /etc/nginx/conf.d/default.conf r,
  owner /etc/nginx/mime.types r,
  owner /etc/nginx/nginx.conf r,
  owner /etc/nsswitch.conf r,
  owner /etc/passwd r,
  owner /proc/1/fd/2 w,
  owner /run/nginx.pid w,
  owner /var/log/nginx/error.log w,

}
```

### Add the Rules by Hand

`aa-logprof` only adds rules for what it asks about. It doesn't ask about network entries, and the rules against an attacker come from the plan, not from the log. I add them by hand.

**Open the profile on the `k8s-worker-1` node**
```bash
sudo vim /etc/apparmor.d/nginx-least-privilege
```

Add these rules before the closing curly brace `}`:
```
  #...
  # Network: nginx listens on TCP port 80.
  network inet stream,

  # Signals between the nginx processes.
  signal (send, receive) peer=nginx-least-privilege,

  # Blocked and logged.
  audit deny /{,usr/}bin/{bash,dash,sh} mrwklx,
  audit deny /usr/share/nginx/html/** wl,
  audit deny /{,var/}run/secrets/** r,
}
```
- `network inet stream`: Allows TCP over IPv4. The log in step 2 shows only `family="inet"`: without the entrypoint script, nginx doesn't listen on IPv6.
- `signal (send, receive) peer=nginx-least-privilege`: Lets the master process stop and reload its worker processes. Skip it if `aa-logprof` already added it.
- `audit deny`: Blocks the action and logs it. A plain `deny` blocks without a log entry.
- `/{,usr/}bin/{bash,dash,sh} mrwklx`: Blocks nginx from starting a shell. In the `nginx` image, `/bin/sh` is a symlink to `/usr/bin/dash`, and AppArmor checks the real path.
- `/usr/share/nginx/html/** wl`: Blocks writes and links in the website.
- `/{,var/}run/secrets/** r`: Blocks the ServiceAccount token, in case a Pod mounts it after all.

The profile is an allowlist. nginx can't start a shell or write to the website, even without the `deny` rules. The `deny` rules keep that protection when someone adds a broad rule later, and they log the attempt.

### Switch to Enforce Mode

`aa-enforce` removes the `complain` mode flag from the file and loads the profile again.

**Switch the profile to enforce mode**
```bash
sudo aa-enforce /etc/apparmor.d/nginx-least-privilege
# Setting /etc/apparmor.d/nginx-least-privilege to enforce mode.
```

**Check the mode of the profile**
```bash
sudo grep nginx-least-privilege /sys/kernel/security/apparmor/profiles
# nginx-least-privilege (enforce)
```

### Fix the Startup Errors

The first test in `enforce` mode fails. The log from step 2 didn't contain all startup actions. It happened in every run. In `enforce` mode, nginx stops at the first action without a rule, and AppArmor logs the block. I fix one error per round.

**Start the test Pod as `hans-dev` user**
```bash
kubectl -n dev apply -f nginx-profile-test.yaml
```

**Check the Pod**
```bash
kubectl -n dev get pod nginx-profile-test
```
```
NAME                 READY   STATUS   RESTARTS     AGE
nginx-profile-test   0/1     Error    1 (8s ago)   9s
```

**Show the error for nginx**
```bash
kubectl -n dev logs nginx-profile-test | grep -E "emerg|alert"
```
```
2026/11/03 10:37:34 [emerg] 1#1: mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```
- `grep -E "emerg|alert"`: Shows only the errors of nginx.

**Show the blocked action on the `k8s-worker-1` node**
```bash
sudo journalctl -k --since "1 min ago" | grep 'apparmor="DENIED"' | grep -oE 'operation=.*'
```
```
operation="mkdir" class="file" profile="nginx-least-privilege" name="/var/cache/nginx/client_temp/" pid=591334 comm="nginx" requested_mask="c" denied_mask="c" fsuid=0 ouid=0
operation="mkdir" class="file" profile="nginx-least-privilege" name="/var/cache/nginx/client_temp/" pid=591729 comm="nginx" requested_mask="c" denied_mask="c" fsuid=0 ouid=0
```

**Add the first rule**
```bash
sudo aa-logprof
```
```
...
Enforce-mode changes:

Profile:  nginx-least-privilege
Path:     /var/cache/nginx/client_temp/
New Mode: owner w
Severity: unknown

 [1 - owner /var/cache/nginx/client_temp/ w,]
...
```

Press `N`, type `/var/cache/nginx/**`, press `A`, and save with `S`. nginx creates more directories there. `aa-logprof` loads the saved profile. You don't need `apparmor_parser -r`.

Repeat these steps until `kubectl logs` shows no `emerg` or `alert` lines. The profile took six rounds.

**Run the test again after each rule**
```bash
kubectl -n dev delete pod nginx-profile-test
kubectl -n dev apply -f nginx-profile-test.yaml
sleep 10
kubectl -n dev get pod nginx-profile-test
kubectl -n dev logs nginx-profile-test | grep -E "emerg|alert"
```

**Add the next rule on the `k8s-worker-1` node**
```bash
sudo aa-logprof
```

| Round | Error in `kubectl logs`                             | Keys                                      | Rule                                 |
| ----- | --------------------------------------------------- | ----------------------------------------- | ------------------------------------ |
| 1     | `mkdir() "/var/cache/nginx/client_temp" failed`     | `N`, type `/var/cache/nginx/**`, then `A` | `owner /var/cache/nginx/** w,`       |
| 2     | `chown("/var/cache/nginx/client_temp", 101) failed` | `A`                                       | `capability chown,`                  |
| 3     | `bind() to 0.0.0.0:80 failed`                       | `2`, then `A`                             | `capability net_bind_service,`       |
| 3     |                                                     | `A`                                       | `owner /var/log/nginx/access.log w,` |
| 3     |                                                     | `2`, then `A`                             | `owner /proc/1/fd/1 w,`              |
| 4     | `open() "/run/nginx.pid" failed`                    | `A`                                       | `owner /run/nginx.pid r,`            |
| 5     | `setgid(101) failed`                                | `4`, then `A`                             | `capability setgid,`                 |
| 6     | `setuid(101) failed`                                | `3`, then `A`                             | `capability setuid,`                 |
- `S`: Saves the profile at the end of every round.
- `chown`: The master process gives the cache directories to the `nginx` user.
- `net_bind_service`: Lets nginx bind port 80.
- `setgid` and `setuid`: Let the master process start its worker processes as the `nginx` user.

For a capability, `aa-logprof` also offers abstractions of other programs, such as `include <abstractions/postfix-common>`. Press the number of the `capability` line instead.

> **_Warning:_** In rounds 5 and 6, the Pod shows `Running` with `0` restarts. The master process runs, but every worker process stops with `setgid(101) failed` or `setuid(101) failed`, and nginx answers no requests. Check `kubectl logs`, not only the status of the Pod.

### Test the Profile

The test Pod from step 2 shows whether nginx works with the final profile in `enforce` mode.

**Create the test Pod again as `hans-dev` user**
```bash
kubectl -n dev apply -f nginx-profile-test.yaml
```

**Check that the Pod runs**
```bash
kubectl -n dev get pod nginx-profile-test
```
```
NAME                 READY   STATUS    RESTARTS   AGE
nginx-profile-test   1/1     Running   0          6s
```

**Forward a local port to the Pod**
```bash
kubectl -n dev port-forward pod/nginx-profile-test 8080:80 &
```

**Request the home page**
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/
# 200
```

**Request a missing page**
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/missing
# 404
```

**Check for blocked actions on the `k8s-worker-1` node**
```bash
sudo journalctl -k --since "1 min ago" | grep 'apparmor="DENIED"' | grep -oE 'operation=.*'
```

The command prints nothing: the profile blocked nothing that nginx needed. If a `DENIED` line shows up, add the missing rule with `aa-logprof`.

**Stop the port forward**
```bash
kill %1
```

**Delete the test Pod**
```bash
kubectl -n dev delete pod nginx-profile-test
# pod "nginx-profile-test" deleted from dev namespace
```

The delete must finish within a few seconds. If the Pod stays in `Terminating`, the `signal` rules are missing.

**Show the final profile on the `k8s-worker-1` node**
```bash
sudo cat /etc/apparmor.d/nginx-least-privilege
```
```
# Last Modified: Tue Nov  3 11:44:52 2026
abi <abi/3.0>,

include <tunables/global>

profile nginx-least-privilege flags=(attach_disconnected) {
  include <abstractions/base>

  capability chown,
  capability dac_override,
  capability net_bind_service,
  capability setgid,
  capability setuid,

  network inet stream,

  signal (receive send) peer=nginx-least-privilege,
  signal receive peer=runc,

  audit deny /usr/share/nginx/html/** wl,
  audit deny /{,usr/}bin/{bash,dash,sh} mrwlkx,
  audit deny /{,var/}run/secrets/** r,

  /usr/share/nginx/html/** r,
  owner /etc/group r,
  owner /etc/nginx/conf.d/ r,
  owner /etc/nginx/conf.d/default.conf r,
  owner /etc/nginx/mime.types r,
  owner /etc/nginx/nginx.conf r,
  owner /etc/nsswitch.conf r,
  owner /etc/passwd r,
  owner /proc/1/fd/1 w,
  owner /proc/1/fd/2 w,
  owner /run/nginx.pid r,
  owner /run/nginx.pid w,
  owner /var/cache/nginx/** w,
  owner /var/log/nginx/access.log w,
  owner /var/log/nginx/error.log w,

}
```

`aa-logprof` sorts the rules and writes `signal (send, receive)` as `signal (receive send)`. The rules work the same.

---

## Step 4: Load the Profile on All Nodes

A Pod with a `Localhost` profile only starts on a node where the profile is loaded. The Deployment can run on any node with the label. Every node needs the profile.

### Copy and Load the Profile

The profile file on `k8s-worker-1` is the source. I copy it to the laptop first, and from there to the control plane nodes.

**Copy the profile file from `k8s-worker-1` to the laptop**
```bash
scp worker-1@k8s-worker-1:/etc/apparmor.d/nginx-least-privilege .
```
- `.`: Saves the file in the current directory on the laptop.

**Install and load the profile on the control plane nodes**
```bash
for node in master-1 master-2 master-3; do
  scp nginx-least-privilege $node@k8s-$node:/home/$node/
  ssh -t $node@k8s-$node sudo mv nginx-least-privilege /etc/apparmor.d/
  ssh -t $node@k8s-$node sudo apparmor_parser -r /etc/apparmor.d/nginx-least-privilege
done
```
- `ssh -t`: Opens a terminal. `sudo` needs it to ask for the password.
- `scp` and `mv`: `scp` can't write to `/etc/apparmor.d` as a normal user. The file goes to the home directory first, and `sudo mv` moves it.

**Check the mode on every control plane node**
```bash
for node in master-1 master-2 master-3; do
  echo -n "$node: "
  ssh -t $node@k8s-$node sudo grep nginx-least-privilege /sys/kernel/security/apparmor/profiles
done
```
```
master-1: nginx-least-privilege (enforce)
master-2: nginx-least-privilege (enforce)
master-3: nginx-least-privilege (enforce)
```
- `echo -n "$node: "`: Prints the node name in front of the result, on the same line.

### Check the Profile After a Reboot

`apparmor.service` loads every file in `/etc/apparmor.d` when the node boots. A reboot shows that the profile survives. I drain the node first, as with any planned reboot.

Think of the node as a shop and the Pods as the customers. A reboot without a drain closes the shop with customers still inside: the Pods stop in the middle of their work. `kubectl drain` first sends all customers outside the shop and then closes the shop. This stops the Pods properly and marks the node unschedulable. No new Pods are scheduled on the node. The Pods start again on the other nodes. After the reboot, run `kubectl uncordon` to open the shop again.

**Drain `k8s-worker-1` node as `hans-adm` user**
```bash
kubectl drain k8s-worker-1 \
  --ignore-daemonsets --delete-emptydir-data \
  --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```

During the drain, the Pods move to the control plane nodes, because they have no taint in my homelab. The `nginx` Deployment from the RBAC project keeps running.

**Check that `k8s-worker-1` node is set to unschedulable**
```bash
kubectl get nodes --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```
```
NAME           STATUS                     ROLES           AGE    VERSION
k8s-master-1   Ready                      control-plane   237d   v1.35.5
k8s-master-2   Ready                      control-plane   237d   v1.35.5
k8s-master-3   Ready                      control-plane   237d   v1.35.5
k8s-worker-1   Ready,SchedulingDisabled   <none>          237d   v1.35.5
```

**Reboot `k8s-worker-1` node**
```bash
ssh -t worker-1@k8s-worker-1 sudo reboot
```
```
Broadcast message from root@k8s-worker-1 on pts/2 (Tue 2026-11-03 12:10:01 CET):

The system will reboot now!

Connection to k8s-worker-1 closed.
```

Wait about a minute until the worker node is back.

**Check the mode after the reboot**
```bash
ssh -t worker-1@k8s-worker-1 sudo grep nginx-least-privilege /sys/kernel/security/apparmor/profiles
# nginx-least-privilege (enforce)
```

**Make `k8s-worker-1` node schedulable again**
```bash
kubectl uncordon k8s-worker-1 --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```

**Check that `k8s-worker-1` node is schedulable again**
```bash
kubectl get nodes --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```
```
NAME           STATUS   ROLES           AGE    VERSION
k8s-master-1   Ready    control-plane   237d   v1.35.5
k8s-master-2   Ready    control-plane   237d   v1.35.5
k8s-master-3   Ready    control-plane   237d   v1.35.5
k8s-worker-1   Ready    <none>          237d   v1.35.5
```

### Label the Nodes

The label tells the scheduler which nodes have the profile. `hans-adm` sets it, because `edit` can't change Nodes.

**Label the nodes as `hans-adm` user**
```bash
kubectl label nodes k8s-master-1 k8s-master-2 k8s-master-3 k8s-worker-1 \
  apparmor-nginx=loaded \
  --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```

**List the nodes with the label**
```bash
kubectl get nodes -l apparmor-nginx=loaded \
  --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```
```
NAME           STATUS   ROLES           AGE    VERSION
k8s-master-1   Ready    control-plane   237d   v1.35.5
k8s-master-2   Ready    control-plane   237d   v1.35.5
k8s-master-3   Ready    control-plane   237d   v1.35.5
k8s-worker-1   Ready    <none>          237d   v1.35.5
```

> **_Note:_** In my homelab, the control plane nodes have no `NoSchedule` taint. The replicas can run on any of the four nodes, and every node needs the profile. On a cluster with the default taint, the Deployment only runs on worker nodes.

---

## Step 5: Enforce Pod Security on the Namespace

RBAC decides who can create a Pod. It doesn't check Pod settings. User `hans-dev` can create Pods in the `dev` Namespace and can also turn AppArmor off with `appArmorProfile.type: Unconfined`. Pod Security stops this. The `baseline` level blocks Pods with `Unconfined`, and still allows `RuntimeDefault` and `Localhost`.

**Enforce baseline on Namespace `dev` as `hans-adm` user**
```bash
kubectl label namespace dev \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  --overwrite \
  --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```
- `enforce=baseline`: Rejects Pods that break the `baseline` standard.
- `enforce-version=latest`: Uses the rules of the current Kubernetes release.

This manifest asks for a Pod without AppArmor:
```bash
cat <<EOF > nginx-unconfined-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: unconfined
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.31.8
      securityContext:
        appArmorProfile:
          type: Unconfined
EOF
```
- `type: Unconfined`: Turns AppArmor off for the container.

**Create the Pod as `hans-dev` user**
```bash
kubectl -n dev apply -f nginx-unconfined-pod.yaml
```
```
Error from server (Forbidden): error when creating "nginx-unconfined-pod.yaml": pods "unconfined" is forbidden: violates PodSecurity "baseline:latest": forbidden AppArmor profile (container "nginx" must not set AppArmor profile type to "Unconfined")
```

The API server rejects the Pod. `hans-dev` can't turn off AppArmor in `dev`.

---

## Step 6: Run nginx as a Deployment

The Deployment runs two replicas with the AppArmor profile, only on nodes with the label. `hans-dev` creates it in the `dev` Namespace.

`kubectl create deployment` can set the command and the port, but not the `securityContext`, the `nodeSelector`, or the token mount. Generate the base manifest with `--dry-run=client -o yaml`, and add the fields.

**Generate the base manifest**
```bash
kubectl -n dev create deployment nginx-server --image=nginx:1.31.8 \
  --replicas=2 --port=80 --dry-run=client -o yaml \
  -- nginx -g "daemon off;" > nginx-server.yaml
```
- `--port=80`: Sets the `containerPort`.
- `-- nginx -g "daemon off;"`: Everything after `--` becomes the `command` of the container.

Edit `nginx-server.yaml` to match this manifest:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-server
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-server
  template:
    metadata:
      labels:
        app: nginx-server
    spec:
      containers:
        - name: nginx
          image: nginx:1.31.8
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 80
          securityContext:
            appArmorProfile:
              type: Localhost
              localhostProfile: nginx-least-privilege
      automountServiceAccountToken: false
      nodeSelector:
        apparmor-nginx: loaded
```
- `command`: Starts `nginx` without the entrypoint script, which the profile would block.
- `appArmorProfile`: Set on the container. The profile applies to every process in it, including processes started with `kubectl exec`.
- `automountServiceAccountToken: false`: Keeps the ServiceAccount token out of the container.
- `nodeSelector`: Schedules the Pods only on nodes with the label `apparmor-nginx=loaded`.

**Create the Deployment as `hans-dev` user**
```bash
kubectl -n dev apply -f nginx-server.yaml
```

**Check the Pods**
```bash
kubectl -n dev get pods -l app=nginx-server -o wide
```
```
NAME               READY   STATUS    RESTARTS   AGE   IP          NODE
nginx-server-...   1/1     Running   0          24s   10.0.1.19   k8s-master-2
nginx-server-...   1/1     Running   0          24s   10.0.0.55   k8s-worker-1
```

> **_Note:_** You can't check the profile with `kubectl exec`. A program such as `ls` starts, but the profile blocks what it can read. Check it on the node instead. The `NODE` column shows where each Pod runs.

SSH into a node where one of the Pods runs.

**Show the nginx processes with the profile on the node of a replica**
```bash
sudo aa-status | grep nginx-least-privilege
```
```
nginx-least-privilege
/usr/sbin/nginx (6861) nginx-least-privilege
/usr/sbin/nginx (6874) nginx-least-privilege
/usr/sbin/nginx (6875) nginx-least-privilege
/usr/sbin/nginx (6876) nginx-least-privilege
/usr/sbin/nginx (6877) nginx-least-privilege
/usr/sbin/nginx (6878) nginx-least-privilege
/usr/sbin/nginx (6879) nginx-least-privilege
```

The first line is the profile. The other lines are the master and worker processes of the replicas on that node.

---

## Step 7: Test the Profile as an Attacker

If an attacker gets access to an nginx Pod or gets a kubeconfig with `kubectl exec` rights, they can open a shell. User `hans-dev` has `kubectl exec` rights in Namespace `dev`. RBAC determines whether the request is allowed, while AppArmor helps control what can happen inside the container.

### Check the Website

The profile must not break the website.

**Forward a local port to the Deployment**
```bash
kubectl -n dev port-forward deployment/nginx-server 8080:80 &
```

**Request the home page**
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/
```
```
200
```

### Try to Start a Shell

A shell is the first thing an attacker needs. With `kubectl exec`, the shell starts because the container runtime starts it. The profile then limits what the shell can do.

**Start a shell in the container**
```bash
kubectl -n dev exec -it deployment/nginx-server -- sh
```
```
# ls
sh: 1: ls: Permission denied
#
```

**Run another program in the container**
```bash
kubectl -n dev exec deployment/nginx-server -- ls /
```
```
ls: cannot open directory '/': Permission denied
command terminated with exit code 2
```

The shell starts, but it can't run other programs like `ls`. A program started directly with `kubectl exec` runs, but can't read `/`.

### Try to Change the Website

An attacker changes a website through a shell or a program in the container.

**Overwrite the home page**
```bash
kubectl -n dev exec deployment/nginx-server -- sh -c 'echo hacked > /usr/share/nginx/html/index.html'
```
```
sh: 1: cannot create /usr/share/nginx/html/index.html: Permission denied
command terminated with exit code 2
```

**Check that the home page didn't change**
```bash
curl -s http://localhost:8080/ | grep -i title
```
```
<title>Welcome to nginx!</title>
```

`sh` starts, but the profile blocks writing. nginx itself only has read access to `/usr/share/nginx/html`.

**Stop the port forward**
```bash
kill %1
```

### Read the Blocked Actions

The `audit deny` rules and the blocked actions show up in the kernel log of the node where the Pod runs. Find the node in the `NODE` column of `kubectl -n dev get pods -l app=nginx-server -o wide`.

**List the blocked actions on the node of the Pod**
```bash
sudo journalctl -k --since "15 min ago" | grep 'apparmor="DENIED"' | grep -oE 'operation=.*'
```
```
operation="open" class="file" profile="nginx-least-privilege" name="/" pid=1158576 comm="ls" requested_mask="r" denied_mask="r" fsuid=0 ouid=0
operation="open" class="file" profile="nginx-least-privilege" name="/dev/tty" pid=1160040 comm="sh" requested_mask="wr" denied_mask="wr" fsuid=0 ouid=0
operation="exec" class="file" profile="nginx-least-privilege" name="/usr/bin/ls" pid=1160070 comm="sh" requested_mask="x" denied_mask="x" fsuid=0 ouid=0
operation="exec" class="file" profile="nginx-least-privilege" name="/usr/bin/ls" pid=1160070 comm="sh" requested_mask="x" denied_mask="x" fsuid=0 ouid=0
operation="open" class="file" profile="nginx-least-privilege" name="/" pid=1160239 comm="ls" requested_mask="r" denied_mask="r" fsuid=0 ouid=0
operation="open" class="file" profile="nginx-least-privilege" name="/usr/share/nginx/html/index.html" pid=1160401 comm="sh" requested_mask="wc" denied_mask="wc" fsuid=0 ouid=0
```
- `grep -oE 'operation=.*'`: Drops the timestamp and the audit header.
- `/dev/tty`: The shell from `kubectl exec -it` tries to open the terminal. The profile blocks it, but the shell still runs.

Each attempt shows up with its path and access. On a production cluster, these entries belong in a log system with an alert: nginx never starts a shell by itself.

---

## Updating the Profile

A new `nginx` image or a new configuration may need new rules. Update the file on `k8s-worker-1` first, and test it there with the test Pod from step 2.

**Switch the profile to complain mode on the `k8s-worker-1` node**
```bash
sudo aa-complain /etc/apparmor.d/nginx-least-privilege
```

`aa-complain` adds the `complain` flag to the file and loads the profile again. The new mode applies to the running containers too. You don't have to restart the Pods.

Run the new image or configuration, and generate the new rules with `aa-logprof` as in [step 3](#generate-the-rules). Switch back with `aa-enforce`, and fix the startup errors as in [step 3](#fix-the-startup-errors). Finally, copy the file to the other nodes as in [step 4](#copy-and-load-the-profile).

> **_Warning:_** In `complain` mode, the profile blocks nothing on that node. Switch back to `enforce` as soon as the test is done.

---

## Cleanup

Remove the profile when you no longer need it, or after a test run. In my homelab, I keep the profile and the Deployment.

### Remove the Profile

A Pod that uses the profile can't start on a node without it. Remove the Pods first, then the label, and then the profile.

Optionally, keep a copy of the profile to compare with the profile of a new run.

**Keep a copy of the profile on the `k8s-worker-1` node**
```bash
sudo cp /etc/apparmor.d/nginx-least-privilege ~/nginx-least-privilege.bak
```

**Delete the Deployment as `hans-dev` user**
```bash
kubectl -n dev delete deployment nginx-server
```

**Remove the label from the nodes as `hans-adm` user**
```bash
kubectl label nodes \
  k8s-master-1 k8s-master-2 k8s-master-3 k8s-worker-1 apparmor-nginx- \
  --kubeconfig ~/.kube/rbac-users/hans-adm.conf
```
- `apparmor-nginx-`: The `-` at the end removes the label.

**Unload and delete the profile on every node**
```bash
for node in master-1 master-2 master-3 worker-1; do
  ssh -t $node@k8s-$node "sudo apparmor_parser -R /etc/apparmor.d/nginx-least-privilege && \
    sudo rm /etc/apparmor.d/nginx-least-privilege"
done
```
- `-R`: Removes the profile from the kernel. Without `rm`, the node loads it again on the next boot.

### Remove the Pod Security Label and the Files

The Pod Security label on the `dev` Namespace and the files on the laptop are only for this project.

**Remove the Pod Security labels as `hans-adm` user**
```bash
kubectl --kubeconfig ~/.kube/rbac-users/hans-adm.conf label namespace dev \
  pod-security.kubernetes.io/enforce- pod-security.kubernetes.io/enforce-version-
```

**Delete the files on the laptop**
```bash
rm nginx-least-privilege nginx-profile-test.yaml nginx-unconfined-pod.yaml nginx-server.yaml
```

### Start Fresh

To run the project again from step 2, remove the profile first, as in [Remove the Profile](#remove-the-profile). Then start with a new syslog on `k8s-worker-1`. `aa-logprof` reads the whole `/var/log/syslog`, including the entries from earlier runs. Without a new file, it asks about the old entries, and the startup errors in step 3 don't show up.

**Move the old syslog aside on the `k8s-worker-1` node**
```bash
sudo mv /var/log/syslog /var/log/syslog.old
```

**Restart rsyslog**
```bash
sudo systemctl restart rsyslog
```
- rsyslog keeps writing to the moved file until it restarts. After the restart, it creates a new `/var/log/syslog`.

**Check that the new syslog has no entries for the profile**
```bash
sudo grep -c nginx-least-privilege /var/log/syslog
# 0
```

---

## Common Errors

| Error                                                                   | Cause                                                       | Fix                                                                       |
| ----------------------------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------- |
| `CreateContainerError` with `apparmor profile not found`                | The profile isn't loaded on the node                        | Load the profile on the node, and check the label                         |
| Pod stays in `Terminating`, `FailedKillPod` with `permission denied`    | The `signal (receive) peer=runc` rule is missing            | Add the rule, and load the profile again                                  |
| Pod in `CrashLoopBackOff` after the switch to `enforce`                 | nginx needs an action that the profile doesn't allow        | Find the `DENIED` entries with `journalctl -k`, and add the rule          |
| `mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)` | The profile misses a rule for an action at startup          | Read the `DENIED` entries, and add the rule with `aa-logprof`             |
| Pod `Running`, but `kubectl logs` shows `setuid(101) failed`            | The worker processes stop, the master process keeps running | Allow `capability setuid,` and `capability setgid,`                       |
| Pod `Pending` with `didn't match Pod's node affinity/selector`          | No node has the label `apparmor-nginx=loaded`               | Load the profile, then label the node                                     |
| `violates PodSecurity "baseline:latest"`                                | The Pod sets `Unconfined`, or another blocked field         | Use `Localhost` or `RuntimeDefault`                                       |
| No `ALLOWED` entries in the log                                         | The `complain` flag is missing, or `--since` is too short   | Check the mode in `/sys/kernel/security/apparmor/profiles`, and `--since` |
| `ERROR: Can't find system log "/var/log/syslog"`                        | `rsyslog` isn't installed                                   | Install `rsyslog`, and run step 2 again                                   |
| `aa-logprof` asks about entries from an earlier run                     | `/var/log/syslog` still has the old entries                 | Start with a new syslog, as in Start Fresh                                |

---

## Conclusion

A strict `securityContext` blocks most of what an attacker wants to do. Start here, because it works for every Pod:

- `runAsNonRoot: true`: The container doesn't run as root.
- `allowPrivilegeEscalation: false`: A process can't become root.
- `capabilities.drop: ["ALL"]`: The container gets no special rights.
- `readOnlyRootFilesystem: true`: The container can only write to its mounted volumes.
- `seccompProfile.type: RuntimeDefault`: Blocks dangerous syscalls.

Kubernetes also gives every container the runtime's default AppArmor profile, `cri-containerd.apparmor.d`, when you don't set one.

Use a custom AppArmor profile for what a `securityContext` can't do:

| Goal                               | `securityContext` | Custom AppArmor profile        |
| ---------------------------------- | ----------------- | ------------------------------ |
| Stop a shell from running programs | Can't block it    | The shell can't start programs |
| Allow one file and block another   | All or nothing    | A rule per path                |
| Log the blocked actions            | Can't log it      | `audit deny`                   |

With this custom profile a shell in the container can't do anything useful. It can't run programs, read most files, or change the website. Most attacks try to start with a shell, and a `securityContext` can't stop the shell from running programs.

You also have to keep a custom profile up to date. This one took six rounds to work, and a new image version can need new rules. Use a custom profile for applications that attackers can reach easily, such as a web server on the internet. For other applications, a strict `securityContext` and the default profile of the runtime are enough.

---

## Reference

| Concept                      | What it means                                                                |
| ---------------------------- | ---------------------------------------------------------------------------- |
| `cri-containerd.apparmor.d`  | The default profile of containerd. Protects the node, not the application.   |
| `complain` mode              | Allows every action, and logs what the profile doesn't allow.                |
| `enforce` mode               | Blocks every action the profile doesn't allow.                               |
| Allowlist profile            | No broad `file,` or `capability,` rule. Only the listed actions are allowed. |
| `audit deny`                 | Blocks an action and logs it. A plain `deny` doesn't log.                    |
| `attach_disconnected`        | Lets the profile handle paths outside the mount namespace of the container.  |
| `signal (receive) peer=runc` | Lets `runc` stop the container.                                              |
| `localhostProfile`           | The profile name from the `profile` line, not the file name.                 |
| `nodeSelector`               | Schedules Pods only on nodes with the profile.                               |
| Pod Security `baseline`      | Rejects `Unconfined`, and allows `RuntimeDefault` and `Localhost`.           |

- [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Enforce Pod Security Standards with Namespace Labels](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [AppArmor: Confine Your Application](https://apparmor.net/getting-started/confine-your-application/)
- My notes: [AppArmor](/notes/kubernetes/cks/2-system-hardening/3-1-apparmor/), [Creating AppArmor Profiles](/notes/kubernetes/cks/2-system-hardening/3-2-creating-apparmor-profiles/), and [AppArmor in Kubernetes](/notes/kubernetes/cks/2-system-hardening/3-3-apparmor-in-kubernetes/).

---

## Best Practices

- Build a profile from the log in `complain` mode, not from memory. Then test it in `enforce` mode: `complain` mode can miss actions at startup.
- Check `kubectl logs` after a profile change, not only the Pod status. A Pod can show `Running` while its worker processes fail.
- Write an allowlist. Don't start from `file,` and add `deny` rules.
- Type paths in `aa-logprof` with `(N)ew`. `(G)lob` can turn `/etc/nginx/*` into `/etc/**`.
- Use `audit deny` for actions that only an attacker needs, and alert on the log entries.
- Load the profile on a node before you label it. Remove the label before you remove the profile.
- Pin the image tag in production. A new image can need new rules.
- Turn off the ServiceAccount token mount when the application doesn't call the Kubernetes API.
- Enforce Pod Security on the Namespace. RBAC can't stop a user from turning AppArmor off.
- On a large cluster, distribute profiles with a tool such as the Security Profiles Operator, instead of `ssh` loops.

---