---
title: TLS Certificate Management
date: 2026-05-10
weight: 5
tags: [Security, TLS, Certificates]
---
# TLS Certificate Management

Securing a Kubernetes cluster requires TLS certificates for every component. Generating certificates from scratch using OpenSSL, inspecting them during a health check, and managing them at scale using the Kubernetes Certificate API.

---

## Generating TLS Certificates

Several tools are available for generating certificates, including EasyRSA, OpenSSL, and CFSSL. This article uses OpenSSL throughout.

### CA Certificates

Start with the Certificate Authority certificates, since all other certificates depend on them.

**Generate a private key**
```bash
openssl genrsa -out ca.key 2048
```

**Generate a Certificate Signing Request (CSR)**
```bash
openssl req -new -key ca.key -subj "/CN=Kubernetes-CA" -out ca.csr
```

The CN field specifies the name of the component the certificate is for. Since this certificate is for the Kubernetes CA, it is named `Kubernetes-CA`.

**Self-sign the certificate**
```bash
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

The CA certificate is self-signed using its own private key. All other certificates will be signed using this CA key pair. The CA now has its private key (`ca.key`) and root certificate (`ca.crt`).

---

### Client Certificates

#### Admin User

The admin user follows the same three-step process.

**Generate a private key**
```bash
openssl genrsa -out admin.key 2048
```

**Generate a CSR**
```bash
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
```

The CN can be any name. It is the name `kubectl` authenticates with and the name that appears in audit logs. The `O`parameter adds the user to the `system:masters` group, which grants administrative privileges on the cluster.

**Sign the certificate using the CA**
```bash
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

Specifying the CA certificate and CA key makes the certificate valid within the cluster. The output is `admin.crt`, which the admin user uses to authenticate to the Kubernetes cluster.

#### System Components

System component certificates must be prefixed with `system` in the CN field:

| Component               | CN Field                         |
|-------------------------|----------------------------------|
| kube-scheduler          | `system:kube-scheduler`          |
| kube-controller-manager | `system:kube-controller-manager` |
| kube-proxy              | `system:kube-proxy`              |

The same three-step process applies to each.

---

### Using Client Certificates

To authenticate using the admin certificate in a REST API call:

```bash
curl https://<kube-api-server>:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

The more common approach is to store these parameters in a `kubeconfig` file, which specifies the API server endpoint, certificates, and other connection details.

---

### CA Root Certificate

Every component must have a copy of the CA root certificate to validate certificates sent by other components. When configuring any server or client with certificates, the CA root certificate must also be specified.

---

### Server Certificates

#### etcd

Generate a certificate for `etcd` following the same three-step process, naming it `etcd-server`. In a high-availability environment, `etcd` runs across multiple servers and requires additional peer certificates to secure communication between `etcd` members.

**Specify the certificates when starting the etcd server**
```bash
--cert-file=/path/to/etcd-server.crt
--key-file=/path/to/etcd-server.key
--peer-cert-file=/path/to/etcd-peer.crt
--peer-key-file=/path/to/etcd-peer.key
--trusted-ca-file=/path/to/ca.crt
--peer-trusted-ca-file=/path/to/ca.crt
```

#### kube-apiserver

The `kube-apiserver` is reachable by many names and aliases:

- `kubernetes`
- `kubernetes.default`
- `kubernetes.default.svc`
- `kubernetes.default.svc.cluster.local`
- The IP address of the host or pod running the API server

All of these names must be present in the certificate as Subject Alternative Names. Create an OpenSSL configuration file to include them:

`openssl.cnf`
```ini
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1  = <kube-api-server-host-ip>
IP.2  = <kube-api-server-pod-ip>
```

Pass this config file when generating the CSR and signing the certificate:

**Generating the CSR**
```bash
openssl req -new -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -config openssl.cnf \
  -out apiserver.csr

openssl x509 -req -in apiserver.csr \
  -CA ca.crt -CAkey ca.key \
  -CAcreateserial \
  -extensions v3_req \
  -extfile openssl.cnf \
  -out apiserver.crt
```

The `kube-apiserver` also acts as a client when communicating with `etcd` and the `kubelet`. All certificate paths are passed into the API server configuration:

```bash
--client-ca-file=/path/to/ca.crt
--tls-cert-file=/path/to/apiserver.crt
--tls-private-key-file=/path/to/apiserver.key

# Certificates for connecting to etcd
--etcd-cafile=/path/to/ca.crt
--etcd-certfile=/path/to/apiserver-etcd-client.crt
--etcd-keyfile=/path/to/apiserver-etcd-client.key

# Certificates for connecting to kubelets
--kubelet-certificate-authority=/path/to/ca.crt
--kubelet-client-certificate=/path/to/apiserver-kubelet-client.crt
--kubelet-client-key=/path/to/apiserver-kubelet-client.key
```

#### kubelet

A certificate and key pair are required for each node in the cluster. Certificates are named after the node: `node01`, `node02`, `node03`.

Specify them in the `kubelet-config.yaml` for each node:

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  x509:
    clientCAFile: /path/to/ca.crt
tlsCertFile: /path/to/node01.crt
tlsPrivateKeyFile: /path/to/node01.key
```

The kubelet also needs client certificates to authenticate to the `kube-apiserver`. The CN must follow this format:

```
system:node:<node-name>
```

For example: `system:node:node01`. Nodes must also belong to the `system:nodes` group, specified using the `O` parameter in the CSR. These client certificates are added to the `kubeconfig` files on each node.

---

### Certificate Summary

| Certificate                    | Type   | CN                               | Group            |
|--------------------------------|--------|----------------------------------|------------------|
| `ca.crt` / `ca.key`            | CA     | `Kubernetes-CA`                  | —                |
| `admin.crt`                    | Client | `kube-admin`                     | `system:masters` |
| `scheduler.crt`                | Client | `system:kube-scheduler`          | —                |
| `controller-manager.crt`       | Client | `system:kube-controller-manager` | —                |
| `kube-proxy.crt`               | Client | `system:kube-proxy`              | —                |
| `etcd-server.crt`              | Server | `etcd-server`                    | —                |
| `apiserver.crt`                | Server | `kube-apiserver`                 | —                |
| `apiserver-etcd-client.crt`    | Client | `kube-apiserver-etcd-client`     | —                |
| `apiserver-kubelet-client.crt` | Client | `kube-apiserver-kubelet-client`  | —                |
| `node01.crt` (per node)        | Server | `node01`                         | —                |
| `node01-client.crt` (per node) | Client | `system:node:node01`             | `system:nodes`   |

---

## Viewing and Inspecting Certificates

When issues are reported with certificates in the cluster, start with a systematic health check of all certificates across the environment. The approach depends on how the cluster was provisioned.

### How the Cluster Was Provisioned

If `kubeadm` was used, control plane components run as static pods and certificates are managed automatically. If the cluster was set up manually, certificates were generated by hand and components run as native OS services. This determines where certificate files and logs are located.

### Building a Certificate Inventory

Start by compiling an inventory of all certificates in the cluster. For each certificate, record:

- The file path on the disk.
- The Common Name (CN).
- Subject Alternative Names (SANs).
- The organisation.
- The issuer.
- The expiration date.

For a `kubeadm` cluster, the API server manifest is the first place to look:

```
/etc/kubernetes/manifests/kube-apiserver.yaml
```

The flags in that manifest list every certificate file the API server uses. Work through them systematically and note the path for each certificate.

### Inspecting Certificates with OpenSSL

**Decode each certificate**
```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

For each certificate, verify the following:

- **Subject (CN)**: must match the expected component identity. For the API server, expect `kube-apiserver`.
- **Subject Alternative Names**: Every name by which the component is reachable must be listed. A missing SAN is a common cause of TLS failures.
- **Validity (Not After)**: An expired certificate will cause components to reject connections.
- **Issuer**: must match the expected CA. In a kubeadm cluster, the Kubernetes CA is named `kubernetes` by convention.

Repeat this process for every certificate in the inventory: the API server, etcd, controller manager, scheduler, and kubelet certificates.

### What to Verify

- The CN matches the expected component identity.
- All required SANs are present, especially for the API server.
- The certificate belongs to the correct organization.
- The issuer is the correct CA.
- The certificate has not expired and is not about to expire.

### Checking Logs

For a `kubeadm` cluster, components run as static pods:

```bash
kubectl logs -n kube-system kube-apiserver-<node-name>
```

For a cluster set up manually, components run as native OS services:

```bash
journalctl -u kube-apiserver
```

If the API server or `etcd` is completely down, `kubectl` will not function. Access component logs directly through the container runtime:

```bash
crictl ps -a
crictl logs <container-id>
```

---

## The Kubernetes Certificate API

As a cluster grows, managing certificates manually becomes unmanageable. Signing every CSR by logging into the master node, running OpenSSL commands, and manually rotating expired certificates does not scale. The Kubernetes Certificate API provides a built-in workflow for submitting and approving certificate signing requests through the Kubernetes API, without direct access to the CA files on the master node.

### The Manual Process

When a new user needs access to the cluster, the process without the Certificate API works as follows:

1. The new user generates a private key on their own machine.
2. Using that key, they produce a CSR with their name embedded in it.
3. They send the CSR to an existing administrator.
4. The administrator logs into the CA server, signs the CSR using the CA private key and root certificate, and produces a signed certificate.
5. The signed certificate is returned to the new user.

When a certificate expires, the same process repeats. At scale, this becomes a bottleneck.

### Using the Certificate API

**Generate a private key and CSR**
```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
```

**Encode the CSR in base64**
```bash
cat jane.csr | base64 -w 0
```

**Create a `CertificateSigningRequest` object**
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
spec:
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
    - client auth
  request: <base64-encoded-csr>
```

**View pending CSRs**
```bash
kubectl get csr
```

**Approve the request**
```bash
kubectl certificate approve jane-csr
```

**Retrieve the signed certificate**
```bash
kubectl get csr jane-csr -o yaml
echo "<base64-string>" | base64 --decode
```

The decoded output is the certificate in PEM format, ready to distribute to the user.

### CertificateSigningRequest Fields

| Field               | Description                                                            |
|---------------------|------------------------------------------------------------------------|
| `signerName`        | Identifies the signer and the category of certificate being requested. |
| `expirationSeconds` | Controls the validity period of the issued certificate.                |
| `usages`            | Defines the intended purpose, such as `client auth`.                   |
| `request`           | The base64-encoded CSR content.                                        |

### Which Component Handles This?

Certificate operations are managed by the Controller Manager, which contains two dedicated controllers:

- `CSR-Approving`: handles the approval workflow.
- `CSR-Signing`: performs the actual signing using the CA key pair.

The Controller Manager's configuration references the CA root certificate and private key, which is why the CA files on the master node are specified at startup.

---

## Conclusion

TLS certificate management in Kubernetes covers three key areas. Certificates are generated with OpenSSL using a consistent three-step process:

- Create a private key
- Generate a CSR
- Issue a signed certificate.

Every component in the cluster, whether acting as a server or a client, needs its own certificate pair signed by the cluster CA.

Inspecting certificates with OpenSSL and verifying the CN, SANs, issuer, and expiry against a complete inventory is the foundation of any certificate health check.

The Certificates API automates certificate signing and rotation, replacing manual CA access with a Kubernetes-native approval workflow managed by the Controller Manager.

---
