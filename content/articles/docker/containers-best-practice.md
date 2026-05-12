---
title: "Containers Security & Best Practices"
date: 2026-01-01
draft: false
tags: ["Containers", "Docker"]
weight: 5
---

# Container Security and Best Practice

Containers are isolated environments; they share the host operating system's kernel. This means that when a container is misconfigured, it can become a serious vulnerability.

If a hacker escapes from a container running as root, they gain root access to the host system and potentially to the entire system and network.

In this article, I will cover essential security best practices for building production-ready container images, focusing on two critical areas:

1. Image security, of what goes into the container.
2. Runtime security, how the container runs.

> **Always treat your container images as public.**

---

## Choose Minimal Base Images

Base image selection improves both security and image size. The standard Ubuntu image is around 80MB, while an Alpine Linux image can be around 5MB.

The Alpine Linux image dramatically reduces image size by using fewer packages, reducing security risks, and speeding up deployment.

When browsing registries (e.g., Docker Hub) for images, **always use official images or verified publishers**. Random images from unknown publishers could contain malware, crypto miners, or backdoors. Official images are maintained, scanned, and trusted.

> **Always use official or verified images from registries.**

---

## Multi-Stage Builds

A multi-stage build separates the build process from the final runtime image. These build stages serve two important purposes:

1. Reducing the image size.
2. Improving security.

When building applications, you need source code, configuration files, and dependencies, but these files shouldn't be included in the final production image.

The first stage contains all the files needed to build the application, and the second stage copies only the compiled application artifact, and all unnecessary files never make it into the distributed image.

```dockerfile
# Stage 1: Generate HTML
FROM ubuntu:24.04 AS builder

WORKDIR /build
COPY generate-status .
RUN chmod +x generate-status && ./generate-status > index.html

# Stage 2: Serve with nginx (unprivileged)
FROM nginxinc/nginx-unprivileged:1.28

COPY --from=builder /build/index.html /usr/share/nginx/html/index.html

EXPOSE 8080
```

The **first stage** uses Ubuntu to generate an HTML file from the bash script, and the **second stage** uses Nginx to copy the HTML file into the Nginx directory to serve as index.html, with no build scripts.

This results in a smaller image, improves security by using an unprivileged image, and cleaner separation of concerns.

> **Implement multi-stage builds to minimize final image size.**

---

## Layer Order

When you modify a file copied early in the Dockerfile and rebuild the image, all subsequent layers will be rebuilt as well.

The solution is to place these files at the bottom of the Dockerfile. Each time you modify a file, only that layer and subsequent layers are rebuilt, saving build time.

### Dockerfile Layering Rules

Here are the layering rules:

1. **Network-intensive operations**: Place them at the top of the Dockerfile.
2. **Lines that never or rarely change**: Place them at the top
3. **Files that change often**: Place them at the bottom
4. **Metadata instructions** (ENV, CMD, etc.): Place them at the bottom

> **Put frequently changed files at the bottom of the Dockerfile**

---

## Docker Ignore File

When you run `docker build .`, the build context (`.`), the directory containing all relevant files, is sent to the Docker daemon to create the image.

Using a remote server to build container images can increase storage requirements and slow the process.

In a `.dockerignore` file, you specify files and directories that are unnecessary for the build, such as `.git`, `build`, or `node_modules`. For example:
```dockerignore
.git
node_modules
*.log
.env
build/
```

This helps reduce the build context size and prevents sensitive data from being included in the build process.

> **Use `.dockerignore` to exclude unnecessary files from the build context**

---

## Running Containers as Non-Root User

By default, containers run as the root user. This is a critical security issue. When using a standard image as the base layer, the root user has full privileges inside this container.

When a hacker gets access to the container, they have root privileges, can install packages, modify system files, and escape the container to install network packages to scan the infrastructure.

To prevent this, always run containers as a non-root user. Many official images provide unprivileged versions in which the container runs as a non-root user.

For custom images, create a non-root user in the Dockerfile:

```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install nginx -y

# non-root user
RUN useradd --create-home appuser
# switch to non-root user
USER appuser

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

By default, the `appuser` is a non-root user; a hacker can't install packages, modify system files, or switch to the root user.

When running the container in interactive mode:
```bash
docker run -it nginx:1.0.0 bash
```

The prompt shows the non-root user `appuser`, and to verify this, try to update the package list:

```bash
appuser@bda064130ffa:/$ apt update
```

This will throw a **Permission denied** error.

> **Use unprivileged base images when available and always create a non-root user.**

---

## Health Checks

Running containers shows an **up** message in the logs, but that doesn't guarantee they're functioning correctly.

With health checks, you can monitor whether the container actually works correctly by running periodic checks.

```dockerfile
# ...
HEALTHCHECK --interval=30s --timeout=3s \
	--start-period=5s --retries=3 \
	CMD curl -f "http://localhost:8080" || exit 1

EXPOSE 8080
# ...
```
- `--interval=30s`: Check every 30 seconds
- `--timeout=3s`: Each check can take max 3 seconds
- `--start-period=5s`: Wait 5 seconds after startup before first check
- `--retries=3`: Container marked unhealthy after three failed checks
- `curl -f`: Return a non-zero exit code when HTTP status code is an error.
- `|| exit 1`: If curl fails, exit with error code 1

This will run every 30 seconds, if it fails after three attempts, it returns an exit code.

When you run `docker ps` you'll see under STATUS the health status:
```
Up About a minute (healthy)
CONTAINER ID   IMAGE                STATUS
755dddfb3f59   nginx-health:1.0.0   Up 12 seconds (healthy)
```

Or inspect the container that will print the status in the terminal:
```bash
docker inspect --format='{{.State.Health.Status}}' <container_id>
# healthy
```

This enables orchestration tools such as Kubernetes to automatically restart unhealthy containers.

> **Add health checks to enable automatic monitoring and recovery of containers**

---

## Docker Security Best Practices Checklist

**Image Configuration:**

- Use specific image tag versions instead of latest to ensure reproducible builds.
- Run as non-root users.
- Implement multi-stage builds to minimize final image size.
- Use official or verified images from registries.
- Choose minimal base images (prefer Alpine).
- Don't install unnecessary packages.
- Scan images for vulnerabilities regularly (using Docker Scout or Trivy).

**Dockerfile Practices:**

- Pin versions for all dependencies
- Order instructions from least to most frequently changed
- Combine RUN commands (`&&`) to reduce layers
- Use `.dockerignore` to exclude unnecessary files
- Never store secrets in images
- Add health checks for production deployments

---

## Production-Ready Container Image

First, create a `generate-status` Bash script that generate an HTML page that shows the date and hostname:

```bash
#!/bin/bash
# Generate a static HTML status page
cat << EOF
<!DOCTYPE html>
<html>
  <head><title>System Status</title></head>
  <body>
    <h1>System Status</h1>
    <p>Generated: $(date)</p>
    <p>Hostname: $(hostname)</p>
  </body>
</html>
EOF
```

Let's consolidate these security best practices into a production-ready Dockerfile that generates HTML from a Bash script and has Nginx serve it as a non-root user, with a health-check status.

```dockerfile
# Stage 1: Generate status HTML
FROM ubuntu:24.04 AS builder

WORKDIR /build
COPY generate-status .
RUN chmod +x generate-status && ./generate-status > index.html

# Stage 2: Production nginx
# unprivileged, runs as non-root
FROM nginxinc/nginx-unprivileged:1.28

# Install wget for health checks
USER root
RUN apt update -y && \
    apt install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
USER nginx

# Copy the generated HTML
COPY --from=builder /build/index.html /usr/share/nginx/html/index.html

# Open default port
EXPOSE 8080

# Health check using wget
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f "http://localhost:8080" || exit 1
```

The Dockerfile shows a multi-stage build, runs as a non-root user with the unprivileged Nginx image, and sends a health check status every 30 seconds.

**Build the container image**:
```bash
docker build -t generate-status:1.0.0 .
```

**Run the container**:
```bash
docker run -d -p 8080:8080 generate-status:1.0.0
```
This will run the container in detached (background) mode.

**Access the page**:
```bash
curl http://localhost:8080
# Or open localhost:8080 in the browser
```

**Check the health status**:
```bash
docker inspect --format='{{.State.Health.Status}}' <container_id>
# healthy
```
The `inspect` command prints the status in the terminal.

Or list all running containers:
```bash
docker ps
```
You should see the status: Up x seconds (healthy).

---

## Conclusion

These practices are the foundation of production-ready containerization. From here, there's always more to improve, but implementing these core principles will enhance container security and protect the infrastructure from common malicious attacks.

- **Run as non-root**: Create a user with `useradd`, and switch to the user
- **Multi-stage builds**: Separate build dependencies from runtime
- **Choose minimal base images**: alpine or slim
- **Scan for vulnerabilities**: Use Docker Scout or Trivy
- **Never include secrets**: Inject at runtime, not build time
- **Add health checks**: Let Docker monitor container health
- **Pin versions**: Base images, packages, and dependencies
- **Use `.dockerignore`**: Exclude unnecessary files from build


---

## Definitions

**Multi-Stage Build**: A Dockerfile feature that separates build-time from runtime for smaller, more secure images.

**Base Image**: The foundation image that the container is built upon.

**Alpine Linux**: Minimal Linux distribution (~5MB) image for containers.

**Non-Root User**: A Linux user without root/administrator privileges that increases security

**unprivileged**: Official images are pre-configured to run as a non-root user (UID 101).

**Health Check**: Docker command that runs periodically inside a container to verify its functioning correctly, enabling automatic recovery by orchestration tools.

**Container Registry**: A repository for storing and distributing container images (e.g., Docker Hub, GitHub Container Registry).

**Docker Layer**: Each instruction in a Dockerfile creates a read-only layer. Layers are cached and reused to speed up builds.

**Build Context**: The directory with its contents (`.`) is sent to the Docker daemon when running `docker build`. Controlled by `.dockerignore`.

**`.dockerignore`**: A file that specifies which files and directories to exclude from the build context, similar to `.gitignore`.

**Official Image**: Docker Hub images verified and maintained by Docker or the software publisher, indicated by a verified checkmark.

---
