---
title: "Dockerfile"
date: 2026-01-30
draft: false
tags: ["Containers", "Docker"]
weight: 4
---

# Dockerfile

A Dockerfile is a text file that defines how to build a container image. It contains a series of instructions that Docker executes in sequence, each creating a new layer in the final image.

The result is a portable, reproducible package that includes the application code, dependencies, libraries, configuration, and a minimal OS filesystem required to run on any system where Docker is installed.

---

## Create Dockerfile

The file must be named `Dockerfile` with a capital **D** and no file extension.

**Create a simple application directory**:
```bash
mkdir -p ~/lab/docker && cd ~/lab/docker
```

**Create the Bash script**:
```bash
cat > greet << 'EOF'
#!/bin/bash
echo "Hello from my container!"
EOF
```
This command concatenates the text between `EOF` into a single file.

**Make the script executable**:
```bash
chmod +x greet
./greet
```

**Create a `Dockerfile` with capital D**:
```bash
vim Dockerfile
```

**Add the basic Dockerfile structure**:
```dockerfile
FROM ubuntu:24.04  # Start with a base image
WORKDIR /app       # Set working directory
COPY greet .       # Copy your application files
CMD ["./greet"]    # Define what runs when container starts
```
This Dockerfile uses Ubuntu 24.04 as the base image, sets `/app` as the working directory, copies the greet script, and specifies the command to run when the container starts.

**Build the image**:
```bash
docker build -t greet:1.0.0 .
```
- `-t`: Tag the image with name `greet` and version `1.0.0`
- `.`: The dot (current directory) specifies the build context. Everything in this directory is sent to the Docker daemon for building.

**Run your new image**:
```bash
docker run greet:1.0.0
```
This command starts the container, executes the `./greet` script, prints "Hello from my container" to the terminal, and exits the container.

It's a reproducible, portable application that can run anywhere with a container runtime.

---

## The Build Process Breakdown

During the build, Docker outputs the build steps. Here's what each section represents:

### Initial Setup Steps

When running `docker build -t greet:1.0.0.` command. Docker reads the Dockerfile from top to bottom, executes each instruction, and sends it to the build engine.

```
[internal] load build definition from Dockerfile 0.1s
=> => transferring dockerfile: 134B
```

Docker contacts Docker Hub to check if there's a newer version of the Ubuntu image and authenticates.
```
[internal] load metadata for docker.io/library/ubuntu:24.044.8s
[auth] library/ubuntu:pull token for registry-1.docker.io0.0s
```

Checks for a `.dockerignore` file.
```
[internal] load .dockerignore0.0s
=> => transferring context: 2B
```

### Dockerfile Instructions (Layers)

Docker builds images layer by layer using the instructions in the Dockerfile. Most instructions create **OverlayFS** layers, while some only add metadata:

```dockerfile
FROM ubuntu:24.04  # Layer 1: Base Ubuntu filesystem
WORKDIR /app       # Layer 2: Set working directory
COPY greet .       # Layer 3: Copy the greet script
CMD ["./greet"]    # Metadata only - no layer created
```

**Layer 1 - FROM**: Pulls and extracts the Ubuntu base image.
```
[1/3] FROM docker.io/library/ubuntu:24.04@sha256:cd1dba... 1.2s
=> => resolve docker.io/library/ubuntu:24.04@sha256:cd1dba...0.0s
=> => sha256:36bf709aa36d... 28.86MB / 28.86MB 0.7s
=> => extracting sha256:36bf709aa36d...0.5s
```

Sends the build context (`.`) to the daemon.
```
[internal] load build context0.0s
=> => transferring context: 112B
```

**Layer 2 - WORKDIR**: Creates the `/app` directory and sets it as the working directory.
```
[2/3] WORKDIR /app 0.1s
```

**Layer 3 - COPY**: Copies the `greet` file into the working directory (`.`).
```
[3/3] COPY greet . 0.0s
```

**CMD Instruction (Metadata)**: The CMD instruction doesn't appear as a build step because it only adds metadata to the image configuration. This happens during the config export phase:
```
=> => exporting config sha256:c13af43... 0.0s
```

### Final Export

Packages everything into the final image and tags it.
```
=> exporting to image0.1s
=> => exporting layers 0.0s
=> => exporting manifest sha256:d4efa86f...0.0s
=> => exporting config sha256:c13af43c...0.0s
=> => naming to docker.io/library/greet:1.0.00.0s
```

Docker uses an **OverlayFS** to stack these layers:
```
┌─────────────────────────────┐
│ Metadata: CMD ["./greet"]   │ ← Not a layer
├─────────────────────────────┤
│ Layer 3:COPY greet .        │ ← Script
├─────────────────────────────┤
│ Layer 2:WORKDIR /app.       │ ← Working
├─────────────────────────────┤ directory
│ Layer 1:FROM ubuntu:24.04   │ ← Base OS
└─────────────────────────────┘
```

Each layer is read-only and contains only the differences from the previous layer.

---

## Dockerfile Instructions

Every instruction in the Dockerfile creates a new layer in the image. These layers are stacked using OverlayFS. This improves efficiency: if a layer hasn't changed, Docker reuses the cached version.

> See my other article about [OverlayFS](https;//hansterhorst/articles/containers/#union-file-system-overlayfs)

### FROM - Base Image Selection

The `FROM` instruction specifies the base image. Always use specific version tags, never `latest`:

```dockerfile
FROM ubuntu:24.04     # Good - specific version
FROM debian:bookworm. # Smaller than Ubuntu
FROM alpine:3.19      # Even smaller (~5MB)
```

Alpine images are particularly useful for minimizing image size.

### RUN - Executing Commands During Build

The `RUN` instruction executes commands during the build process. For example, to install curl:

```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install curl -y
WORKDIR /app
COPY greet .
CMD ["./greet"]
```

Rebuild the image:
```bash
docker build -t greet:1.1.0 .
```

Build process logs:
```
=> [2/4] RUN apt update && apt install curl -y
```

Now curl is available inside the container:
```bash
docker run greet:1.1.0 which curl
# /usr/bin/curl
```

#### Understanding Layers and the OverlayFS

Each instruction in the Dockerfile creates an image layer, which represents a filesystem. Layers are immutable and stack on top of one another to form the final image.

Combine multiple commands in a single `RUN` instruction to minimize layers:
```dockerfile
# Bad - creates three separate layers
RUN apt update
RUN apt install -y curl
RUN apt clean

# Good - creates one layer
RUN apt update && \
apt install -y curl && \apt clean \
```
- `&&`: Use the AND operator to chain commands.

#### Including Required Dependencies

Containers are minimal and include all packages required to run the application. If you need another package, you can install it. For example, if the script uses curl:

```bash
docker run --rm ubuntu:24.04 bash -c "
apt update &&apt install curl -y &&curl https://example.com"
```

### COPY - Copy Files

`COPY` copy files from the build context into the image:

```bash
# Copy single file
COPY start /app/

# Copy directory contents
COPY scripts/ /app/scripts/

# Copy multiple files
COPY config.json data.json /app/

# Copy with a different name
COPY config.prod.json /app/config.json
```

### WORKDIR - Setting the Working Directory

`WORKDIR` sets the working directory for subsequent instructions. Use `WORKDIR` instead of `RUN cd`:

```dockerfile
WORKDIR /app
```
This will be the current directory.

Copies the greet script in the working directory:
```
COPY greet .
```
- `.`: Is the working directory

### ENV - Environment Variables

Set environment variables at runtime:
```dockerfile
ENV API_URL=https://api.example.com
ENV LOG_LEVEL=debug
```

### EXPOSE Ports

`EXPOSE` Open ports on the container:
```dockerfile
EXPOSE 8080
```
It doesn't actually publish ports

You need `-p` flag to expose the ports:
```bash
docker run -p 8090:8080 greet
```
- `-p`: Port to expose
- `8090:8080`:
- `8090`: Port on the host
- `8080`: Port on the container

### CMD - Default Command

`CMD` specifies the command to run when the container starts:

```dockerfile
CMD ["./greet"]
```

You can override CMD at runtime:

```bash
docker run greet:1.1.0 date # Runs date instead of greet
```

### CMD vs ENTRYPOINT

`ENTRYPOINT` is similar to `CMD` but harder to override, making it ideal for containers that act as executables.

Create a backup script in the Docker directory:

```bash
mkdir ~/lab/docker && cd ~/lab/docker
```

Create the `backup` script:
```bash
cat > backup << 'EOF'
#!/bin/bash
case "$1" in
create)
	echo "Creating backup..."
	;;
	restore)
		echo "Restoring backup..."
		;;
	list)
		echo "Listing backups..."
		;;
	*)
		echo "Usage: backup {create|restore|list}"
		exit 1
		;;
esac
EOF
```

Make the script executable:
```bash
chmod +x backup
```

Create the Dockerfile:
```dockerfile
FROM ubuntu:24.04
WORKDIR /app
COPY backup .
RUN chmod +x backup
ENTRYPOINT ["./backup"]
CMD ["list"]
```

Build and test:
```bash
docker build -t backup:1.0.0 .
docker run backup:1.0.0         # Runs "list" by default
docker run backup:1.0.0 create  # Runs "create"
docker run backup:1.0.0 restore # Runs "restore"
```

The `CMD` provides a default argument to `ENTRYPOINT`. Arguments passed to `docker run` replace the `CMD` value but are passed to the `ENTRYPOINT`.

To override the entrypoint entirely:
```bash
docker run --entrypoint bash backup:1.0.0
# With shell (-it)
docker run --entrypoint bash -it backup:1.0.0
```

> **Rule of thumb**:
> Use `CMD` for most applications.
> Use `ENTRYPOINT` when the container itself is the command (like CLI tools).

---

## Docker Ignore File

When you run `docker build .`, the build context (`.`), the directory containing all its files, is sent to the Docker daemon. Including unnecessary files increases storage usage and slows build time, especially when using remote build servers.

Use a `.dockerignore` file to specify which files to exclude from the build context:
```
.git
node_modules
*.log
.env
build/
```

This improves build speed and prevents the accidental inclusion of sensitive data, such as credentials or API keys, in the image.

---

## Layer Caching and Optimization

Layer caching is one of Docker's most powerful features for speeding up builds, but it requires a strategic approach to Dockerfile organization to maximize its benefits.

When Docker builds an image, it checks whether it has already executed an identical instruction with the same context. If so, it reuses the cached layer instead of rebuilding.

### The Problem: Poor Layer Ordering

When a layer changes, Docker must rebuild it with all subsequent layers, even if those layers haven't changed.

```dockerfile
FROM ubuntu:24.04.   # Layer 1
COPY greet .         # Layer 2 CHANGED THIS FILE
RUN apt update && \  # Layer 3 Rebuild (even unchanged)
apt install curl -y 
WORKDIR /app         # Layer 4 Rebuild (even unchanged)
CMD ["./greet"]
```

Every time you change `greet`, Docker reinstalls all packages because the `COPY` instruction is different from the cache.

### The Solution: Optimize Layer Order

Order instructions from the least frequently changed to the most frequently changed.

```dockerfile
FROM ubuntu:24.04    # Changes: Never
RUN apt update && \  # Changes: Rarely
apt install curl -y 
WORKDIR /app.        # Changes: Rarely
COPY greet .         # Changes: Often
CMD ["./greet"]      # Changes: Sometimes
```

Now, changing greet affects only the `COPY` layer and below. The package installation layer remains cached.

Order the Dockerfile from the least frequently changed to the most frequent change:

1. Base image
2. System dependencies
3. Application dependencies
4. Application code
5. Runtime configuration

Check image layer history:

```bash
docker history greet:1.0.0
```
```
IMAGE CREATEDCREATED BYSIZE
c13af43c... 2 minutes agoCMD ["./greet"] 0B
a8f3d21b... 2 minutes agoCOPY greet . 112B
...
```

---

## Image Tagging and Versioning

Use semantic versioning (SemVer) for your images: `MAJOR.MINOR.PATCH`

- `PATCH`: Backward-compatible bug fixes
- `MINOR`: New features, backward-compatible
- `MAJOR`: Breaking changes

Example workflow:
```bash
# Initial version
docker build -t greet:1.0.0 .

# PATCH, bug fix
docker build -t greet:1.0.1 .

# MINOR, new feature
docker build -t greet:1.1.0 .

# MAJOR, breaking change
docker build -t greet:2.0.0 .
```

Create convenience tags:
```bash
docker build -t greet:1.1.0 -t greet:1.1 -t greet:1 -t greet:latest .
```

You can also use Git commit hashes as tags:
```bash
docker build -t greet:$(git rev-parse --short HEAD) .
```

> **Never** rely on the `latest` in production. **Always specify exact versions**.

---

## Build Arguments (variables)

Pass variables at build time:
```dockerfile
ARG UBUNTU_VERSION=24.04
FROM ubuntu:${UBUNTU_VERSION}
```

Built an image with a specific OS version:
```bash
docker build --build-arg UBUNTU_VERSION=22.04 -t greet:1.0.0 .
```

### Rebuilding Without Cache

Force rebuild all layers:
```bash
docker build --no-cache -t greet:1.0.0 .
```

---

## Container Registries

When you run `docker run nginx`, Docker pulls the image from Docker Hub. When using another registry, such as GitHub Container Registry (ghcr.io), you must specify the full URL.

### Pushing to GitHub Container Registry

First, create a Personal Access Token (PAT) on GitHub:

1. Go to GitHub Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
2. Generate a new token (classic)
3. Select scopes: `write:packages` and `read:packages`
4. Generate and copy the token

Log in to GitHub Container Registry:
```bash
export CR_PAT=YOUR_TOKEN_HERE
echo $CR_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

Tag your image with the registry URL:
```bash
docker tag backup:1.0.0 ghcr.io/YOUR_GITHUB_USERNAME/backup:1.0.0
```

Push the image:
```bash
docker push ghcr.io/YOUR_GITHUB_USERNAME/backup:1.0.0
```

The image is now in your GitHub packages (initially private). To make it public:

1. Go to your GitHub profile → Packages
2. Click on your package
3. Package settings → Change visibility → Public

Now, anyone can pull your image:
```bash
docker pull ghcr.io/YOUR_GITHUB_USERNAME/backup:1.0.0
docker run ghcr.io/YOUR_GITHUB_USERNAME/backup:1.0.0 list
```

You've created a containerized CLI application that anyone in the world can use.

---

## Conclusion

Understanding Dockerfiles is fundamental to containerization. By organizing instructions strategically and leveraging layer caching, you build efficient, reproducible images.

From here, I will explore multi-stage builds to optimize image size, implement security best practices with non-root users, and create a Docker project with all the best practices I learned

---

## Definitions

**Dockerfile**: A text file containing instructions to build a container image.

**Base Image**: The starting point for the new image, containing the operating system and basic utilities.

**Layer**: Each Dockerfile instruction creates a read-only layer in the final image

**Build Context**: The directory with its contents (`.`) is sent to the Docker daemon when running `docker build`.

**Image Tag**: A label to specify the name and version of an image.

**Semantic Versioning (SemVer)**: Version numbering format `MAJOR.MINOR.PATCH` where:
- MAJOR = breaking changes
- MINOR = new features
- PATCH = bug fixes.

**Container Registry**: A repository for storing and distributing container images (e.g., Docker Hub, GitHub Container Registry).

**OverlayFS**: A union filesystem that stacks layers efficiently, allowing Docker to share common layers between images efficiently.

**Layer Caching**: Docker's optimization that reuses previously built layers when nothing has changed, speeding up builds.

**`.dockerignore`**: A file to specify which files to exclude from the build context, similar to `.gitignore`.

**`ENTRYPOINT`**: Dockerfile instruction that defines the executable that always runs when the container starts. Arguments can still be passed.

**`CMD`**: Dockerfile instruction that provides default arguments to ENTRYPOINT or defines the default command if no ENTRYPOINT is set.

**`ARG`**: Build-time variable that can be passed with `--build-arg`.

**`ENV`**: Runtime environment variable that persists in the running container.

---


