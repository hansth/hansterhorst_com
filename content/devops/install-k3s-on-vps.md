---
title: Install K3s on Ubuntu VPS
date: 2025-10-14
tags: ["devops", "k3s", "ubuntu"]
draft: false
weight: 2
---

After completing the **TWN DevOps Bootcamp**, my next goal was to set up my own **DevOps environment** using the tools I learned during the bootcamp. In the previous project, I configured a **secure Ubuntu VPS** to prepare it for running a **K3s Kubernetes cluster**.

**K3s** is a **lightweight, certified Kubernetes distribution** that's easy to install, resource-efficient, and perfect for small clusters, or development environments. By deploying K3s on an Ubuntu VPS, I will have a reliable environment for experimenting with Kubernetes and running applications in containers.

In this project, I will explain the steps I took to set up a single-node K3s cluster on an Ubuntu VPS.


## Why I chose K3s

There are many ways to run Kubernetes, but K3s stood out because:

- **Lightweight**: Minimal resource usage, perfect for small VPS setups.
- **Easy Setup**: One-line installation script.
- **Production-Ready**: Despite being small, it’s stable enough for real-world applications.
- **Built-in Components**: Comes with **containerd**, **Traefik**, and a **local storage** provider.


## Update UFW Firewall

Before installing K3s, I need to update the UFW firewall to open ports that K3s use for networking.

**Login into the server**:
```sh
ssh <USERNAME@<SERVER_IP>
```

**Open the following ports**:
```sh
sudo ufw allow 6443
sudo ufw allow from 10.42.0.0/16 to any
sudo ufw allow from 10.43.0.0/16 to any
```
- `ufw allow 6443/tcp`: Allows communication with the Kubernetes control plane.
- `ufw allow from 10.42.0.0/16 to any`: Enables pod-to-pod and pod-to-service communication.
- `ufw allow from 10.43.0.0/16 to any`: Allows service discovery and inter-service networking.

Without these rules, pods are not able to reach the API server or communicate with each other, breaking essential Kubernetes functionality.


## Install K3s

K3s provides an installation script that automatically installs and configures it as a `systemd` service. This script is available at [https://get.k3s.io](https://get.k3s.io/).

**Switch to the `root` user**:
```sh
sudo su -
```

**Update the Ubuntu package list**:
```sh
apt update && apt upgrade -y
```

**Run the K3s install script**:
```sh
curl -sfL https://get.k3s.io | sh -
```

This script will:
- Download and install K3s.
- Configure it as a `systemd` service.
- Install additional CLI tools such as `kubectl`, `crictl`, and `ctr`.
- Create utility scripts like `k3s-killall.sh` and `k3s-uninstall.sh`
- Generate a Kubernetes configuration file at `/etc/rancher/k3s/k3s.yaml`.
- Start the Kubernetes control plane automatically.

**Check if the service is running**:
```sh
systemctl status k3s.service
```
```output
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-08-31 15:20:26 UTC; 33s ago
       Docs: https://k3s.io
    Process: 5704 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.se>
    Process: 5706 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
    Process: 5712 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 5714 (k3s-server)
      Tasks: 79
     Memory: 1.2G (peak: 1.2G)
        CPU: 37.759s
     CGroup: /system.slice/k3s.service
			    ...
```

**Copy the `k3s.yaml` config file to the user home directory**:
```sh
cp /etc/rancher/k3s/k3s.yaml /home/USERNAME/
```
I need this config file to communicate with the cluster when using an external host.

**Exit as `root` user**:
```sh
exit
```

**Change the owner permission**:
```sh
sudo chown USERNAME:USERNAME /home/USERNAME/k3s.yaml
```
This will change the `root` permission to the user.

**Exit the server**:
```sh
exit
```


## Configure `kubectl` Locally

To interact with the K3s cluster, I first needed to install a command-line tool called `kubectl` and then securely transfer the `k3s.yaml` configuration file to my local environment.

`kubectl` is a command-line tool for communicating with Kubernetes clusters, it allows deploying, manage, and troubleshoot applications within the cluster.

The `k3s.yaml` file, generated during installation, acts as the Kubernetes configuration file that connects `kubectl` to the cluster.

I'm using a Mac, and the most used package manager is **[Homebrew](https://brew.sh)**. For other OS systems see the [Kubernetes documentation](https://kubernetes.io/docs/tasks/tools/).

**Install `kubectl` cli tool**:
```sh
brew install kubernetes-cli
```

**Create a `.kube` directory**:
```sh
mkdir ~/.kube
```
This is the default location where `kubectl` will look for the config file.

**Secure copy the config file to local computer**:
```sh
scp USERNAME@SERVER_IP:/home/USERNAME/k3s.yaml ~/.kube/
```

**Rename the config file to `config`**:
```sh
mv ~/.kube/k3s.yaml ~/.kube/config
```
This is the default filename that `kubectl` will use.

**Change the permission** (to secure the kubeconfig file):
```bash
chmod 600 ~/.kube/config
```

To interact with the cluster, you have to change the server ip-address in the `config` file to the server IP-address.

**Open the config file**:
```sh
sudo vim ~/.kube/config
```

**Change the permission**:
```bash
chmod 600 ~/.kube/config
```

**Change `https://127.0.0.1:6443` to the server ip-address**:
```vim
apiVersion: v1
clusters:
- cluster:
    ...
    server: https://<SERVER_IP>:6443
  name: ...
```

After saving the config file, you can communicate with the cluster.

**Check the connection**:
```sh
kubectl get nodes
```
```
NAME       STATUS   ROLES                  AGE   VERSION
hostname   Ready    control-plane,master   12m   v1.33.4+k3s1
```

**Display all running pods**:
```sh
kubectl get pods -A
```
```output
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-64fd4b4794-lpwl4                  1/1     Running     0          13m
kube-system   helm-install-traefik-4v5br                0/1     Completed   2          13m
kube-system   helm-install-traefik-crd-gr4pj            0/1     Completed   0          13m
kube-system   local-path-provisioner-774c6665dc-n56jz   1/1     Running     0          13m
kube-system   metrics-server-7bfffcd44-z49mw            1/1     Running     0          13m
kube-system   svclb-traefik-09095a6f-4cw56              2/2     Running     0          12m
kube-system   traefik-c98fdf6fb-txnd7                   1/1     Running     0          12m
```


## Deploy a test Application

To confirm everything is working fine, I deployed a simple NGINX pod.

**Create a Deployment**:
```sh
kubectl create deployment nginx --image=nginx
```

**Create a Service to expose the Deployment externally**:
```sh
kubectl expose deployment nginx --port=80 --type=NodePort
```

**Verify the assigned port**:
```sh
kubectl get service nginx
```
This will show details about the `nginx` Service with the NodePort `30476`:
```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP        19m
nginx        NodePort    10.43.190.177   <none>        80:30476/TCP   4m46s
```

Visit `http://<SERVER_IP>:30476` in your browser, you should see the default **NGINX welcome page** confirming that your K3s cluster is running successfully.


## Conclusion

Setting up a **K3s cluster on Ubuntu** turned out to be a straightforward process. By opening the right firewall ports, running the installation script, and configuring `kubectl` locally, I now have a fully functional **Kubernetes environment** for experimenting with containerized applications and practicing real-world **DevOps workflows**.