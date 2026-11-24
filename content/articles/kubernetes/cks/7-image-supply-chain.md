---
title: Image Supply Chain
date: 2026-11-17
weight: 7
draft: false
reviewed: true
tags:
  - CKS
  - Security
  - Kubernetes
  - Supply Chain
  - Trivy
  - Cosign
  - Tutorial
---
# Project Image Supply Chain

In this project, I secure the path of a container image from the registry to a Pod in my homelab cluster. Before the change, any Pod could run any image from any registry, signed or not, with any number of known vulnerabilities. After the change, a Pod in the protected Namespace only runs images from my own GitHub registry, and only when I signed them.

A cluster trusts every image it pulls. A compromised base image, a typosquatted name, or a tag that someone pushed over runs with the same rights as a real image. Scanning shows what is inside an image. Signing proves who built it. An admission policy makes the cluster check both before a Pod starts.

---

## The Plan

I use the frontend image of my [DevOps CI/CD Pipeline](https://github.com/hansth/k8s-gitops) project as an example. The pipeline builds it and pushes it to GitHub Container Registry (GHCR). This project starts where the pipeline stops, and doesn't change the pipeline.

I scan the image and compare it with the default `nginx` image. Then I copy it to a separate GHCR package, sign it, and add two admission policies that reject unsigned images and images from other registries.

The project uses these names and values:

| Item             | Value                                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Source image     | [`ghcr.io/hansth/todo-frontend:0.4.0`](https://github.com/users/hansth/packages/container/package/todo-frontend), built by the pipeline    |
| Project image    | [`ghcr.io/hansth/cks-supply-chain:0.4.0`](https://github.com/users/hansth/packages/container/package/cks-supply-chain), new public package |
| Comparison image | `nginx` from Docker Hub                                                                                                                    |
| Signing key      | `cosign.key` and `cosign.pub` on the laptop                                                                                                |
| Registry policy  | ValidatingAdmissionPolicy `allowed-registries`                                                                                             |
| Signature policy | Kyverno ImageValidatingPolicy `verify-signed-images`                                                                                       |
| Test Namespace   | `secure`, with the label `supply-chain=enforced`                                                                                           |

> **_Note:_** The policies apply only to the `secure` Namespace. The system Pods in `kube-system` pull their images from `registry.k8s.io`, so a cluster-wide policy would block them.

---

## Step 1: Scan the Images

**Trivy** compares the packages in an image against databases of known vulnerabilities. A smaller base image has fewer packages, which means fewer vulnerabilities to patch.

**Scan the default nginx image**
```bash
trivy image --severity HIGH,CRITICAL nginx
```
```
2026-11-17T17:33:09+01:00	INFO	[vuln] Vulnerability scanning is enabled
2026-11-17T17:33:09+01:00	INFO	[secret] Secret scanning is enabled
2026-11-17T17:33:09+01:00	INFO	[secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2026-11-17T17:33:09+01:00	INFO	[secret] Please see https://trivy.dev/docs/v0.76/guide/scanner/secret#recommendation for faster secret detection
2026-11-17T17:33:10+01:00	INFO	Detected OS	family="debian" version="13.7"
2026-11-17T17:33:10+01:00	INFO	[debian] Detecting vulnerabilities...	os_version="13" pkg_num=146
2026-11-17T17:33:10+01:00	INFO	Number of language-specific files	num=0
2026-11-17T17:33:10+01:00	WARN	Using severities from other vendors for some vulnerabilities. Read https://trivy.dev/docs/v0.76/guide/scanner/vulnerability#severity-selection for details.

Report Summary

┌─────────────────────┬────────┬─────────────────┬─────────┐
│       Target        │  Type  │ Vulnerabilities │ Secrets │
├─────────────────────┼────────┼─────────────────┼─────────┤
│ nginx (debian 13.7) │ debian │       124       │    -    │
└─────────────────────┴────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


nginx (debian 13.7)

Total: 124 (HIGH: 123, CRITICAL: 1)
```
- `--severity`: Shows only the vulnerabilities with these levels.

**Scan the frontend image**
```bash
trivy image --severity HIGH,CRITICAL ghcr.io/hansth/todo-frontend:0.4.0
```
```
2026-11-17T17:34:57+01:00	INFO	[vuln] Vulnerability scanning is enabled
2026-11-17T17:34:57+01:00	INFO	[secret] Secret scanning is enabled
2026-11-17T17:34:57+01:00	INFO	[secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2026-11-17T17:34:57+01:00	INFO	[secret] Please see https://trivy.dev/docs/v0.76/guide/scanner/secret#recommendation for faster secret detection
2026-11-17T17:34:58+01:00	INFO	Detected OS	family="alpine" version="3.24.2"
2026-11-17T17:34:58+01:00	INFO	[alpine] Detecting vulnerabilities...	os_version="3.24" repository="3.24" pkg_num=71
2026-11-17T17:34:58+01:00	INFO	Number of language-specific files	num=0
2026-11-17T17:34:58+01:00	WARN	Using severities from other vendors for some vulnerabilities. Read https://trivy.dev/docs/v0.76/guide/scanner/vulnerability#severity-selection for details.

Report Summary

┌───────────────────────────────────────────┬──────┬───────────────┬───────┐
│                  Target                   │ Type │Vulnerabilities│Secrets│
├───────────────────────────────────────────┼──────┼───────────────┼───────┤
│ghcr.io/hansth/todo-...:0.4.0(alpine3.24.2)│alpine│      19       │   -   │
└───────────────────────────────────────────┴──────┴───────────────┴───────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


ghcr.io/hansth/todo-frontend:0.4.0 (alpine 3.24.2)

Total: 19 (HIGH: 18, CRITICAL: 1)
```

The frontend image has fewer vulnerabilities for three reasons in its `Dockerfile`:

- A multi-stage build: Node.js and the npm packages stay in the build stage. Only the built files go into the final image.
- An Alpine base: `nginx:stable-alpine` has far fewer packages than the default Debian-based `nginx`.
- `apk upgrade`: Installs the latest fixes of the Alpine packages at build time.

> **_Note:_** A distroless image goes further and has no shell and no package manager. An attacker who gets into the container has fewer tools to work with.

---

## Step 2: Create an SBOM

A **Software Bill of Materials** (SBOM) lists every package in an image and its version. When a new vulnerability appears, you check the SBOM instead of pulling and scanning each image again.

**Generate a CycloneDX SBOM of the image**
```bash
trivy image --format cyclonedx --output todo-frontend-sbom.json ghcr.io/hansth/todo-frontend:0.4.0
```
- `--format cyclonedx`: Writes the SBOM in the CycloneDX format.
- `--output`: The file for the SBOM.

**Count the packages in the SBOM**
```bash
jq '.components | length' todo-frontend-sbom.json
```
```
72
```

**Scan the SBOM for vulnerabilities**
```bash
trivy sbom --severity HIGH,CRITICAL todo-frontend-sbom.json
```
```
2026-11-17T18:11:20+01:00	INFO	[vuln] Vulnerability scanning is enabled
2026-11-17T18:11:20+01:00	INFO	Detected SBOM format	format="cyclonedx-json"
2026-11-17T18:11:20+01:00	INFO	Detected OS	family="alpine" version="3.24.2"
2026-11-17T18:11:20+01:00	INFO	[alpine] Detecting vulnerabilities...	os_version="3.24" repository="" pkg_num=71
2026-11-17T18:11:20+01:00	INFO	Number of language-specific files	num=0
2026-11-17T18:11:20+01:00	WARN	Using severities from other vendors for some vulnerabilities. Read https://trivy.dev/docs/v0.76/guide/scanner/vulnerability#severity-selection for details.

Report Summary

┌─────────────────────────────────────────┬────────┬─────────────────┐
│                 Target                  │  Type  │ Vulnerabilities │
├─────────────────────────────────────────┼────────┼─────────────────┤
│ todo-frontend-sbom.json (alpine 3.24.2) │ alpine │       19        │
└─────────────────────────────────────────┴────────┴─────────────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


todo-frontend-sbom.json (alpine 3.24.2)

Total: 19 (HIGH: 18, CRITICAL: 1)
```

The result matches the scan of the image in step 1. Trivy only needs the package list, not the image.

---

## Step 3: Copy and Sign the Image

A signature binds an image digest to a key. A tag, such as `0.4.0`, may point to a different image later, whereas a digest always identifies the same content.

### Install crane

[`crane`](https://github.com/google/go-containerregistry) is a command-line tool for container registries from Google, and it talks to the registry API directly, without Docker or a daemon.

I use `crane` to copy the image so the digest remains unchanged. The `crane copy` command copies the image without changes, and the copy keeps the original digest. Docker can't do this with this image. The pipeline stores the image along with a record of how it was built. `docker pull` only fetches the image, and `docker push` then creates a new digest. `crane digest` shows the digest of a tag without pulling the image.

**Install crane on macOS**
```bash
brew install crane
```

On Linux, download the binary from the [GitHub releases](https://github.com/google/go-containerregistry/releases).

**Save the latest release version**
```bash
VERSION=$(curl -s "https://api.github.com/repos/google/go-containerregistry/releases/latest" | jq -r '.tag_name')
```

**Download the release for Linux on x86_64**
```bash
curl -sL "https://github.com/google/go-containerregistry/releases/download/${VERSION}/go-containerregistry_Linux_x86_64.tar.gz" > go-containerregistry.tar.gz
```

**Extract only crane to /usr/local/bin**
```bash
sudo tar -zxvf go-containerregistry.tar.gz -C /usr/local/bin/ crane
```

**Check the installed version**
```bash
crane version
```
```
0.22.1
```

### Copy the Image to Its Own Package

Cosign stores the signature in the registry as an extra tag next to the image. Copy the image to its own package first. The signatures and test tags of this project then stay out of the package of the pipeline.

The login needs a classic personal access token with the `write:packages` scope. GHCR doesn't accept fine-grained tokens.

**Save the access token in a shell variable**
```bash
GHCR_TOKEN=ghp_s5mdSR0PwE1zor...QbDbZ4X4wNv3sb8Ah
```

This is a temporary token, deleted after the project, to show what a classic token looks like.

**Log in to GHCR with your access token**
```bash
echo $GHCR_TOKEN | crane auth login ghcr.io -u hansth --password-stdin
```
- `$GHCR_TOKEN`: A shell variable with the personal access token.
- `--password-stdin`: Keeps the token out of the shell history. 

Cosign uses the same login.

**Copy the image to the project package**
```bash
crane copy ghcr.io/hansth/todo-frontend:0.4.0 ghcr.io/hansth/cks-supply-chain:0.4.0
```
```
2026/11/17 19:10:26 Copying from ghcr.io/hansth/todo-frontend:0.4.0 to ghcr.io/hansth/cks-supply-chain:0.4.0
2026/11/17 19:10:28 mounted blob: sha256:22e5a8a110ec69c6af30325ccfe8bb0bfb94e5783117808061bae5115e1e3dcb
...
2026/11/17 19:10:30 mounted blob: sha256:2c42a4b5e32c31bc2cf476035a9968c791b9c91dd8b1b1a9b22ee1644bcf472b
2026/11/17 19:10:31 ghcr.io/hansth/cks-supply-chain:0.4.0: digest: sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0 size: 2823
```

The `mounted blob` lines show that GHCR reuses the layers of the original.

**Show the digest of the copy**
```bash
crane digest ghcr.io/hansth/cks-supply-chain:0.4.0
```
```
sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0
```

**Show the digest of the original**
```bash
crane digest ghcr.io/hansth/todo-frontend:0.4.0
```
```
sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0
```

Both digests are the same. The copy is the exact image that the pipeline built.

**Save the digest and the image reference with the digest**
```bash
DIGEST=sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0
IMAGE=ghcr.io/hansth/cks-supply-chain@$DIGEST
```

GHCR creates a new package as private. Make it public before you sign it: in the package settings on GitHub, under **Danger Zone**, select **Change package visibility**. The nodes can then pull the image, and Kyverno can read the signature, without credentials. With a private package, Cosign asks if you want to upload to the public transparency log.

> **_Note:_** The copy keeps the `org.opencontainers.image.source` label of the pipeline, and GHCR links the new package to the `k8s-gitops` repository because of it. Links on the package page can lead to the repository settings.

### Sign the Image

Cosign signs the digest and pushes the signature to the same package.

**Install cosign**
```bash
brew install cosign
```

**Generate a key pair**
```bash
cosign generate-key-pair
```

Cosign asks for a password and writes `cosign.key` and `cosign.pub`:
```
Enter password for private key:
Enter password for private key again:
Private key written to cosign.key
Public key written to cosign.pub
```

> **_Warning:_** `cosign.key` signs images with your name. Anyone with the key and its password can sign a malicious image that the cluster accepts. Keep it out of Git, and store it in a password manager.

**Sign the image with the private key**
```bash
cosign sign --key cosign.key $IMAGE
```
```
	The sigstore service, hosted by sigstore a Series of LF Projects, LLC, is provided pursuant to the Hosted Project Tools Terms of Use, available at https://lfprojects.org/policies/hosted-project-tools-terms-of-use/.
	Note that if your submission includes personal data associated with this signed artifact, it will be part of an immutable record.
	This may include the email address associated with the account with which you authenticate your contractual Agreement.
	This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later, and is subject to the Immutable Record notice at https://lfprojects.org/policies/hosted-project-tools-immutable-records/.

By typing 'y', you attest that (1) you are not submitting the personal data of any other person; and (2) you understand and agree to the statement and the Agreement terms at the URLs listed above.
Are you sure you would like to continue? [y/N] y
Enter password for private key:
Signing artifact... \ Pushing signature to: ghcr.io/hansth/cks-supply-chain
```

> **_Note:_** Cosign writes a record of the signature to the public Rekor transparency log, and asks you to confirm first. `cosign verify` checks that the record exists. With a key, the record holds the digest, the signature, and the public key, not your email address.

**Verify the signature with the public key**
```bash
cosign verify --key cosign.pub $IMAGE
```
```
Verification for ghcr.io/hansth/cks-supply-chain@sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key

[{"critical":{"identity":{"docker-reference":"ghcr.io/hansth/cks-supply-chain@sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0"},"image":{"docker-manifest-digest":"sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0"},"type":"https://sigstore.dev/cosign/sign/v1"},"optional":{}}]
```

The output shows the digest that the key signed. The cluster runs the same check in step 5.

Cosign doesn't change the image. It stores the signature as an extra tag in the same package, named after the digest.

**Show the signatures stored for the image**
```bash
cosign tree $IMAGE
```
```
📦 Supply Chain Security Related artifacts for an image: ghcr.io/hansth/cks-supply-chain@sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0
└── 🔗 https://sigstore.dev/cosign/sign/v1 artifacts via OCI referrer: ghcr.io/hansth/cks-supply-chain@sha256:fa23d21ff57f998ccb90bd9987c312bb9f0a840a886923b7c5e63d8b83b9bb56
   └── 🍒 sha256:673ccce29eab66e7445a5c4cfa1778be5451b8c58931bbdee51b542e5911b443
```

The signature lives in `cks-supply-chain`, next to the image. Step 6 shows why that matters.

---

## Step 4: Allow Only Your Registry

A ValidatingAdmissionPolicy checks objects with a Common Expression Language (CEL) expression before the API server stores them. It's built into Kubernetes and needs no extra software.

CEL is a small expression language for checks that returns `true` or `false`, and the API server approves or rejects the request.

**Create the Namespace**
```bash
kubectl create namespace secure
```

**Add the label that the binding selects**
```bash
kubectl label namespace secure supply-chain=enforced
```

The policy rejects every container image that doesn't come from my GHCR account:

```bash
cat <<'EOF' > allowed-registries.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: allowed-registries
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: >-
        object.spec.containers.all(c, c.image.startsWith('ghcr.io/hansth/')) &&
        (!has(object.spec.initContainers) ||
         object.spec.initContainers.all(c, c.image.startsWith('ghcr.io/hansth/')))
      message: "Images must come from ghcr.io/hansth."
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: allowed-registries
spec:
  policyName: allowed-registries
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        supply-chain: enforced
EOF
```
- `failurePolicy: Fail`: Rejects the Pod when the expression can't be validated.
- `matchConstraints`: The requests the policy checks: creating and updating Pods.
- `expression`: The CEL check. Every container and init container image must start with `ghcr.io/hansth/`. The trailing slash matters: without it, an account such as `hansth-evil` passes too.
- `validationActions: [Deny]`: Rejects the request. `Audit` and `Warn` only report it.
- `namespaceSelector`: Applies the policy only to Namespaces with the label.

**Apply the policy and the binding**
```bash
kubectl apply -f allowed-registries.yaml
```

> **_Note:_** The policy checks `containers` and `initContainers` because an attacker could run any image as an init container. 

---

## Step 5: Allow Only Signed Images

Kubernetes can't verify Cosign signatures by itself. Kyverno runs as an admission webhook and checks the signature in the registry before the Pod starts.

### What Is Kyverno

Kyverno is a policy engine for Kubernetes. Policies are defined as Kubernetes resources in YAML, using CEL expressions to perform checks. No separate policy language is required.

Kyverno runs as Pods in the `kyverno` Namespace, and registers itself as an admission webhook. The API server sends every matching request to Kyverno before it stores the object in etcd. Kyverno compares the request with the policies and allows or rejects it.

A Kyverno policy can do more than allow or reject a request:

| Policy type  | What it does                                         | Example                                                 |
| ------------ | ---------------------------------------------------- | ------------------------------------------------------- |
| Validate     | Allows or rejects a request                          | Reject Pods without resource limits                     |
| Mutate       | Changes the object before the API server stores it   | Add a default `securityContext`                         |
| Generate     | Creates extra objects                                | Add a default-deny NetworkPolicy to every new Namespace |
| Verify image | Checks signatures and attestations in the registry   | Reject images without a Cosign signature                |

### Why Kyverno Is Useful Here

Kyverno also uses CEL, like the policy in step 4. Both policies are written the same way, and you only need to learn one expression language.

The ValidatingAdmissionPolicy in step 4 only reads the Pod object. It can check the image name but can't query the registry for a signature. Kyverno can look up each image's signature in GHCR and verify it against the public key.

> **_Note:_** The CKS exam doesn't test Kyverno. It tests the built-in admission control: `ImagePolicyWebhook`, the admission plugins, and ValidatingAdmissionPolicy. Kyverno is a common way to enforce signatures in real clusters.

### Install Kyverno

Kyverno comes as a Helm chart.

**Add the Kyverno Helm repository**
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
```

**Update the Helm repositories**
```bash
helm repo update
```

**Install Kyverno**
```bash
helm install kyverno kyverno/kyverno --version 3.10.0 -n kyverno --create-namespace
```
```
NAME: kyverno
LAST DEPLOYED: Fri Nov 17 20:27:34 2026
NAMESPACE: kyverno
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
NOTES:
Chart version: 3.10.0
Kyverno version: v1.20.0

Thank you for installing kyverno! Your release is named kyverno.

The following components have been installed in your cluster:
- CRDs
- Admission controller
- Reports controller
- Cleanup controller
- Background controller


⚠️  WARNING: Setting the admission controller replica count below 2 means Kyverno is not running in high availability mode.


⚠️  WARNING: PolicyExceptions are disabled by default. To enable them, set '--enablePolicyException' to true.
⚠️  WARNING: The legacy kyverno.io policy types are deprecated and will be removed in a future release. Migrate to their policies.kyverno.io replacements:
    - ClusterPolicy / Policy → ValidatingPolicy, MutatingPolicy, GeneratingPolicy, ImageValidatingPolicy (and their namespaced variants)
    - ClusterCleanupPolicy / CleanupPolicy → DeletingPolicy / NamespacedDeletingPolicy
    - PolicyException (kyverno.io) → PolicyException (policies.kyverno.io)
    See https://kyverno.io/docs/guides/migration-to-cel/ for the migration guide.

💡 Note: There is a trade-off when deciding which approach to take regarding Namespace exclusions. Please see the documentation at https://kyverno.io/docs/installation/#security-vs-operability to understand the risks.
```

> **_Note:_** The chart runs one admission controller Pod. The policy rejects requests when Kyverno can't be reached. When that one Pod is down, no Pod can start in the Namespace `secure`. Set the replica count to 3 for high availability in a production cluster.

**Check that the Kyverno Pods are running**
```bash
kubectl -n kyverno get all
```
```
NAME                                                READY   STATUS    RESTARTS   AGE
pod/kyverno-admission-controller-758c695576-88k4k   1/1     Running   0          66s
pod/kyverno-background-controller-86969647b-n9v5d   1/1     Running   0          66s
pod/kyverno-cleanup-controller-678f9f5466-d498w     1/1     Running   0          66s
pod/kyverno-reports-controller-9c95f666c-6x7dt      1/1     Running   0          66s

NAME                                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/kyverno-background-controller-metrics   ClusterIP   10.108.127.68    <none>        8000/TCP   66s
service/kyverno-cleanup-controller              ClusterIP   10.100.168.173   <none>        443/TCP    66s
service/kyverno-cleanup-controller-metrics      ClusterIP   10.102.219.85    <none>        8000/TCP   66s
service/kyverno-reports-controller-metrics      ClusterIP   10.106.73.104    <none>        8000/TCP   66s
service/kyverno-svc                             ClusterIP   10.111.177.201   <none>        443/TCP    66s
service/kyverno-svc-metrics                     ClusterIP   10.97.76.58      <none>        8000/TCP   66s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kyverno-admission-controller    1/1     1            1           66s
deployment.apps/kyverno-background-controller   1/1     1            1           66s
deployment.apps/kyverno-cleanup-controller      1/1     1            1           66s
deployment.apps/kyverno-reports-controller      1/1     1            1           66s

NAME                                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/kyverno-admission-controller-758c695576   1         1         1       66s
replicaset.apps/kyverno-background-controller-86969647b   1         1         1       66s
replicaset.apps/kyverno-cleanup-controller-678f9f5466     1         1         1       66s
replicaset.apps/kyverno-reports-controller-9c95f666c      1         1         1       66s
```

### Create the Signature Policy

The policy accepts an image from my GHCR account only if it has a valid signature from the key. Paste the content of `cosign.pub` in the `data` field:

```bash
cat <<'EOF' > verify-signed-images.yaml
apiVersion: policies.kyverno.io/v1
kind: ImageValidatingPolicy
metadata:
  name: verify-signed-images
spec:
  validationActions: [Deny]
  webhookConfiguration:
    timeoutSeconds: 15
  evaluation:
    background:
      enabled: false
  matchConstraints:
    namespaceSelector:
      matchLabels:
        supply-chain: enforced
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  matchImageReferences:
    - glob: "ghcr.io/hansth/*"
  attestors:
    - name: cosign
      cosign:
        key:
          data: |
           -----BEGIN PUBLIC KEY-----
           MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEiFKb08s5ELCmwNR9RIjIvckqf5OC
           D0xNx9XZz8+sRZ0OrleQ0jvUFoEzGd0JltoJWUmDYE4wKYEV6W5T4GzyoQ==
           -----END PUBLIC KEY-----
  validations:
    - expression: >-
        images.all(type, images[type].all(image,
          verifyImageSignatures(image, [attestors.cosign]) > 0))
      message: "The image isn't signed with the homelab key."
EOF
```
- `validationActions: [Deny]`: Rejects a Pod with an unsigned image.
- `matchConstraints`: Checks new and changed Pods in the Namespaces with the label.
- `matchImageReferences`: The images to verify: every image of my GHCR account, not only the project package. Images from other registries are left to the policy in step 4.
- `attestors.cosign.key.data`: The public key. The private key stays private!
- `expression`: Checks every image in the Pod: containers and init containers. `images` groups the images by container type, and `all` loops through each group.
- `verifyImageSignatures`: Counts the valid signatures of an image. Every image needs at least one.

**Apply the policy**
```bash
kubectl apply -f verify-signed-images.yaml
```

---

## Step 6: Test the Policies

Three Pods show what each policy does. Each one tests a different gap.

**Run the signed image from the project package**
```bash
kubectl -n secure run signed --image=$IMAGE --env=BACKEND_ORIGIN=http://localhost:22111
```
```
pod/signed created
```
- `--env=BACKEND_ORIGIN`: The nginx configuration of the frontend proxies `/api/` to this address. nginx needs a host name it can resolve at startup. The default `host.docker.internal` doesn't exist in the cluster.

Both policies accept the Pod. The image comes from my GHCR account, and the signature is valid.

**Run an image from Docker Hub**
```bash
kubectl -n secure run dockerhub --image=nginx:alpine
```
```
The pods "dockerhub" is invalid: : ValidatingAdmissionPolicy 'allowed-registries' with binding 'allowed-registries' denied request: Images must come from ghcr.io/hansth.
```

The ValidatingAdmissionPolicy rejects the Pod, before Kyverno checks it.

**Run the unsigned image of the pipeline**
```bash
kubectl -n secure run unsigned --image=ghcr.io/hansth/todo-frontend:0.4.0 --env=BACKEND_ORIGIN=http://localhost:22111
```
```
Error from server: admission webhook "ivpol.validate.kyverno.svc-fail-finegrained-verify-signed-images" denied the request: Policy verify-signed-images failed: The image isn't signed with the homelab key.; verification details: image "ghcr.io/hansth/todo-frontend:0.4.0@sha256:219a6c71382e16f1c927bd63d43d50a59a6bb2669af454ba8b1dc1b847b620c0", attestor "cosign": failed to verify cosign signatures: no signatures found
```

The registry check passes: the image comes from my GHCR account. The image even has the same digest as the signed copy. But Cosign stores the signature in the repository of the image, `cks-supply-chain`. Kyverno looks for a signature in `todo-frontend`, finds none, and rejects the Pod. Every new version that the pipeline pushes gets the same result, until a signing step is added to the pipeline.

### ImagePolicyWebhook for the Exam

The CKS exam uses the built-in `ImagePolicyWebhook` admission plugin instead of Kyverno. The API server sends every image to an external backend, and the backend allows or rejects it.

The admission configuration points to a kubeconfig file for the backend:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/policywebhook/kubeconf
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false
```
- `kubeConfigFile`: The kubeconfig with the URL and certificates of the backend.
- `allowTTL` and `denyTTL`: How long the API server caches an answer, in seconds.
- `retryBackoff`: The time between retries, in milliseconds.
- `defaultAllow: false`: Rejects every Pod when the backend can't be reached. With `true`, an attacker who stops the backend turns off the check.

Add both flags to `/etc/kubernetes/manifests/kube-apiserver.yaml`, and mount the directory with a `hostPath` volume, as in [Secret Encryption at Rest](/articles/kubernetes/cks/5-secret-encryption-at-rest/):

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/policywebhook/admission_config.yaml
```

> **_Note:_** I don't run `ImagePolicyWebhook` in my homelab: it needs a backend service that I don't have. With `defaultAllow: false` and no backend, no Pod can start.

---

## Cleanup

The policies only apply to the Namespace `secure`. Delete the Namespace to remove the test Pods.

**Delete the test Namespace**
```bash
kubectl delete namespace secure
```

Keep the policies and Kyverno if you want to use the Namespace label for real workloads. Otherwise, remove them.

**Delete the policies**
```bash
kubectl delete -f verify-signed-images.yaml -f allowed-registries.yaml
```

**Uninstall Kyverno**
```bash
helm uninstall kyverno -n kyverno
```

To remove the image and its signature, delete the `cks-supply-chain` package in the package settings on GitHub. The `todo-frontend` package of the pipeline stays as it is.

**Remove the GHCR login from the laptop**
```bash
crane auth logout ghcr.io
```

---

## Common Errors

| Error                                                        | Cause                                                              | Fix                                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
| The signed Pod is rejected by Kyverno                        | The Pod uses the tag, and the tag points to another digest         | Run the Pod with the digest from `crane digest`                    |
| The nodes can't pull the image                               | The GHCR package is still private                                  | Make the package public, or add an `imagePullSecret`               |
| The signed Pod starts, then restarts                         | nginx can't resolve the host in `BACKEND_ORIGIN`                   | Set `BACKEND_ORIGIN` to a host that exists, such as `localhost`    |
| Kyverno rejects every Pod with a timeout                     | Kyverno can't reach the registry                                   | Check the network from the Kyverno Pods to `ghcr.io`               |
| The Docker Hub Pod still starts                              | The Namespace has no label                                         | Add the label `supply-chain=enforced` to the Namespace             |
| `cosign verify` fails with `no matching signatures`          | The image was signed with another key, or not at all               | Sign the digest again with `cosign.key`                            |

---

## Conclusion

Each control closes a different gap. Scanning shows the known vulnerabilities, and a multi-stage build on a small base image reduces them. The SBOM keeps the package list for later checks. The registry policy stops images from unknown sources. The signature policy proves that I signed the image, and that nobody changed it after I signed it.

The unsigned pipeline image shows the next step: signing belongs in CI, right after the push. Then every release is signed without a manual step.

None of these controls stops a vulnerability that nobody knows about yet. A signed image with a bug in my own code is still a signed image. Runtime security, the next domain, watches what the container does after it starts.

---

## Reference

| Concept                        | What it means                                                               |
| ------------------------------ | --------------------------------------------------------------------------- |
| `trivy image --severity`       | Scans an image and shows only the chosen severity levels                    |
| SBOM                           | The list of every package in an image, with its version                     |
| `trivy sbom`                   | Scans an SBOM file instead of an image                                      |
| `crane copy`                   | Copies an image from registry to registry, with the same digest             |
| Image digest                   | The `sha256` hash of the image content. It never changes                    |
| `cosign sign --key`            | Signs an image digest with a private key                                    |
| `cosign verify --key`          | Checks the signature of an image with the public key                        |
| ValidatingAdmissionPolicy      | A built-in admission check with a CEL expression                            |
| ImageValidatingPolicy          | A Kyverno policy that verifies image signatures                             |
| `ImagePolicyWebhook`           | A built-in admission plugin that asks an external backend about each image  |

- [Trivy: SBOM](https://trivy.dev/latest/docs/supply-chain/sbom/)
- [Cosign: Signing Containers](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)
- [Kyverno: ImageValidatingPolicy](https://kyverno.io/docs/policy-types/image-validating-policy/)
- [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Admission Controllers: ImagePolicyWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)

---

## Best Practices

- Use a multi-stage build and a minimal base image, such as Alpine or distroless.
- Scan images in CI, and fail the build on HIGH and CRITICAL vulnerabilities.
- Keep an SBOM for every image you release.
- Sign the digest, never the tag, and run Pods by digest.
- Allow only the registries you control.
- Check init containers and ephemeral containers, not only containers.
- Keep the signing key out of Git, and protect it with a strong password.

---

## CKS Exam Takeaways

- `trivy image --severity HIGH,CRITICAL <IMAGE>` scans an image. Use it to find the Pods with vulnerable images and delete them.
- `ImagePolicyWebhook` needs `--enable-admission-plugins` and `--admission-control-config-file` on the API server, plus a `hostPath` volume for the configuration.
- `defaultAllow: false` rejects Pods when the backend is down.
- The admission configuration has `kind: AdmissionConfiguration`. `ImagePolicyWebhook` is the plugin name.
- A ValidatingAdmissionPolicy needs a ValidatingAdmissionPolicyBinding. Without the binding, it does nothing.
- Prefer image digests to tags. A tag can change, a digest can't.

---
