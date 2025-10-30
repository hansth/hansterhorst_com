---
title: Environment Variables in K8s Manifests
date: 2025-10-14
tags: ["devops", "k3s", "env", "security"]
draft: false
weight: 4
---

In my projects where I use Kubernetes, I use **environment variables** to keep sensitive data and server-specific values **out of the GitHub repository** while allowing the manifests to remain reusable across environments (development, staging, and production).


## Why Environment Variables

Instead of hard-coding configuration values like file paths, hostnames, or credentials directly inside Kubernetes manifests, I define them in a local `.env` file. During deployment, these variables will be **automatically injected** into the manifests using [`envsubst`](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html) command.

This approach provides:

- **Security:** No secrets, hostnames, and other credentials are stored in version control.
- **Portability:** The same manifests can be used across different environments.
- **Automation:** Deployment can be fully scripted.

> **_Note_**: For sensitive data like passwords, API keys, or certificates, use **Kubernetes Secrets** or a **secret manager**, like HashiCorp Vault, AWS Secrets Manager. Environment variables are best for non-sensitive configuration values.

## Defining the Variables

All variables are defined in a local `.env` file in the root project directory.

**Create a `.env` file in the root project with variables**:
```bash
ROOT_DOMAIN=example.dev
WWW_ROOT_DOMAIN=www.example.dev
EMAIL=dev@email.com
NGINX=nginx
```

**After editing, load the environment variables into the shell**:
```bash
set -a 
source .env 
set +a
```
- `set -a`: Enables automatic export mode for all variables.
- `source .env`: Loads all variables into the current environment.
- `set +a`: Disables automatic export mode when done.

Without `set -a`, variables would only be available in the current shell and not to scripts or subprocesses.

**Verify**:
```bash
echo $NGINX
```

> **Always add `.env` file to the `.gitignore` to keep it private**.


## Using Variables in Kubernetes Manifests

Instead of hard-coding values, use placeholders like `${VARIABLE_NAME}` in the manifest.

**Example ClusterIssuer**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: "${EMAIL}" # replace for your valid email
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - selector: {}
        http01:
          ingress:
            class: traefik
```

This makes the manifests reusable across multiple environments with minimal changes.

## Deploying with `envsubst`

To apply manifests that include environment variables, use the `envsubst` command to substitute variables before applying them with `kubectl`.

**Apply the manifest file**:
```bash
envsubst < MANIFEST_FILE.yaml | kubectl apply -f -
```
- `envsubst`: Utility that replaces environment variable with their actual values.
- `< MANIFEST_FILE.yaml`: Redirects the manifest file as input to `envsubst`.
- `|`: Pipes the output from `envsubst` to the next command.
- `kubectl apply`: Creates or updates Kubernetes resources
- `-f -`: Read the file from (`-`) standard input (stdin) instead of a file.

If you want to substitute **only certain variables**, you can specify them explicitly:
```bash
envsubst '${EMAIL},${ROOT_DOMAIN}' < MANIFEST_FILE.yaml | kubectl apply -f -
```

This ensures the correct values are dynamically inserted into your resources during deployment.


## Conclusion

Using environment variables is **ideal for non-secret configuration values** and helps ensure **consistency and reusability** across development, staging, and production.

Here are the steps I took to implement it:

1. **Created a `.env` file** with environment-specific values such as domain names and emails.
2. **Loaded variables** into the shell using `set -a` and `source .env`.
3. **Referenced variables** in manifest files using `${VARIABLE_NAME}` placeholders.
4. **Used `envsubst`** to dynamically substitute and apply the manifests with real values.

By following this workflow, I achieved a **more secure, flexible, and automated deployment process**, allowing the manifests to stay clean, version-controlled, and environment-agnostic.