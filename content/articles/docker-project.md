---
title: "Docker Project"
date: 2026-01-30
draft: false
tags: ["Containers", "Docker"]
weight: 6
---

# Docker Project

After completing the [KubeCraft](https://www.skool.com/kubecraft) Containers Fundamentals course, I built a multi-container application using the concepts I learned. I implemented best practices and security, created a production-ready Dockerfile, and learned to use Docker Compose for orchestration.

---

## The project: A Bad Joke Generator

The application displays a joke in the browser every 30 seconds. This seems simple, but it demonstrates key concepts of a multi-container setup:

- **Backend Service** that fetches data from an external API
- **Frontend Service** that displays the HTML page
- **Shared Storage** between the containers
- **Dockerfile** creates the backend container image
- **Docker Compose** file that handles the orchestration between the two containers

The architecture of the application:

```
          Backend Service      Frontend Service. 
          ┌───────────────┐    ┌──────────────┐  
          │    updater    │    │    nginx     │  
          │    (bash)     │    │              │  
┌─────┐   │               │    │              │  
│ API │◀︎──┼─ fetch jokes  │    │  serve HTML  │  
└─────┘   │  write HTML   │    │              │  
          └───────────────┘    └──────▲───────┘  
                  │                   │          
                  └─────▶︎ /html ◀︎─────┘         
                      Shared Storage)            
```

- The **backend service** fetches a joke from the external API every 30 seconds and writes the generated HTML page to the shared storage.
- The **frontend service** retrieves the HTML page from shared storage and renders the page in the browser.
- The **shared storage** is a Docker Volume that mounts the `/html` directory to Nginx's `/html` directory.

This demonstrates how two containers communicate without a direct network connection.

---

## Initial Setup

First, create the project directory:

```bash
mkdir bad-joke-generator && cd bad-joke-generator
```

I will create three files:

- `compose.yaml`: Docker Compose file for defining the services and the volume
- `Dockerfile`: Build the custom container image for the backend
- `fetch-joke`: Bash script that fetches a joke and generates the HTML page

## Building the Bash Script

The Bash script forms the heart of the application. Every 30 seconds, it fetched a joke from an external API and generated the HTML page to write to the shared storage.

**Create the `fetch-joke` script**:
```bash
#!/bin/bash
# Fetch bad jokes and update the HTML page

API_URL="https://icanhazdadjoke.com/"
HTML_FILE="/html/index.html"

echo "Starting fetching jokes..."

while true; do
  # Fetch a random bad joke
  JOKE=$(curl -s "$API_URL" -H "Accept: text/plain")

  # Generate HTML
  cat > "$HTML_FILE" << EOF
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Bad Joke Generator</title>
  <style>
    body {
      font-family: sans-serif;
      max-width: 600px;
      margin: 50px auto;
      padding: 20px;
      text-align: center;
    }
    .joke {
      font-size: 1.5em;
      margin: 40px 0;
      padding: 20px;
      background: #f0f0f0;
      border-radius: 10px;
    }
    .meta {
      color: #666;
      font-size: 0.9em;
    }
  </style>
</head>
<body>
  <h1>Bad Joke Generator</h1>
  <div class="joke">$JOKE</div>
  <p class="meta">Updated: $(date)</p>
  <p class="meta">Refreshes every 30 seconds</p>

  <script>
    setTimeout(() => {
      location.reload();
    }, 30000);
  </script>
</body>
</html>
EOF
  echo "Updated joke: $JOKE"
  sleep 30
done
```

Make it executable:
```bash
chmod +x fetch-joke
```

Test the script:
```bash
./fetch-joke
```

> **_Note_**: You'll see that it fetches a joke and tries to write it to `/html/index.html`, but the directory doesn't exist yet. This will be created when the Docker Volume is defined in the Docker Compose file.

---

## Creating the Dockerfile

Create a `Dockerfile` with this initial version:

```dockerfile
FROM ubuntu:24.04

RUN apt update -y && apt install curl -y && \
    rm -rf /var/lib/apt/lists/*

COPY fetch-joke /fetch-joke
RUN chmod +x /fetch-joke

CMD ["/fetch-joke"]
```

This Dockerfile:

- Uses Ubuntu as the base image
- Installs `curl` for API requests
- Copies and runs the fetch-joke script

> **_Note_**: I haven't implemented multi-stage builds or non-root users yet. This is a proof-of-concept.

---

## What is Docker Compose?

Docker Compose is an orchestration tool for defining and running multiple containers using a single YAML configuration file.

When running multiple related containers individually with `docker run`, it becomes difficult to manage, error-prone, hard to replicate, and difficult to share with team members. Instead of running containers individually, you define them in a Docker Compose file.

This allows you to manage the entire application stack, including backend and frontend services, networking, and volumes, in a single YAML file.

---

### Defining Services with Docker Compose

Create `compose.yaml`:

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - html:/html

  frontend:
    image: nginx:1.29-alpine
    ports:
      - "8080:80"
    volumes:
      - html:/usr/share/nginx/html:ro

volumes:
  html:
```

Let's break down this configuration:

**Backend service**:
- Builds from the custom `Dockerfile` in the current directory (`.`)
- Mounts a volume named `html` at `/html` directory where the script writes to
- The `Dockerfile` will create the custom image

**Frontend service**:
- Uses the official Nginx image
- Maps the host port `8080` to the container port `80`
- Mounts the `html` volume at Nginx's default serving directory
- The `:ro` flag makes it read-only for Nginx, not write

**Volumes**:\
The volume is the critical part: the **backend service** writes the generated HTML file, and the **frontend service** retrieves it. This decoupling allows each service to do its specific task.

```yaml
volumes:
  html:
```
- Docker Compose creates a named volume `html` with default settings that both services use.

The trailing colon in YAML is a syntax for an empty mapping.


## Running the Application

Run both containers with a single command:
```bash
docker compose up -d --build
```
- `up`: Start all the services
- `-d`: Runs the containers in detached mode (background).
- `--build`: Ensures the image is rebuilt whenever code changes.

Verify that the containers are running:
```bash
docker ps
```

You should see two containers named `backend-1` and `frontend-1`. Docker Compose automatically prefixes container names with the directory name and appends service names.
```
CONTAINER ID  IMAGE                        STATUS         NAMES
c3aea9b2af66  nginx:1.28                   Up 10 seconds  bad_jokes_generator-frontend-1
6aab4d3fdc56  bad_jokes_generator-backend  Up 10 seconds  bad_jokes_generator-backend-1
```

Check the logs:
```bash
docker compose logs bad_jokes_generator-backend-1 -f
```
- `-f`: Follows the log output

You'll see a joke being fetched every 30 seconds.

## Accessing the Application

Test locally with URL:
```bash
curl localhost:8080
```
You should see HTML with a bad joke. To view it in the browser, go to `http://localhost:8080`

### SSH Tunnel

When running this on another device in the network, you need to tunnel the port on a remote host. With **SSH local port forwarding**, also called SSH tunneling, you create a secure tunnel between the local machine and a remote server:
```bash
ssh -L 8080:localhost:8080 <YOUR_REMOTE_HOST>
```
- `-L`: Enables local port forwarding
- `8080`: The first port on the **local machine**
- `localhost`: The destination host **from the remote server's perspective**
- `8080`: The second port on the destination host
- `<YOUR_REMOTE_HOST>`: The SSH server to connect to

Now open `http://localhost:8080` in the browser. You'll see the joke website with jokes refreshing every 30 seconds.

---

## Docker Compose Commands

Docker Compose provides several useful commands:

**Stop all services**:
```bash
docker compose down
```

**Stop and remove volumes**:
```bash
docker compose down -v
```

**Rebuild images**:
```bash
docker compose up -d --build
```

**View logs for a specific service**:
```bash
docker compose logs -f backend
```

**Execute commands in a service**:
```bash
# Show the current user
docker compose exec frontend whoami
# Interact with the built-in bash shell
docker compose exec backend bash
# Test the connection
docker compose exec frontend nginx -t
# Interact with the built-in sh shell
docker compose exec -it frontend sh
```

The advantage over plain `docker` commands is that you reference services by name rather than looking up container IDs.

---

## Implementing Production Best Practices

The initial setup works, but it has serious security issues:

- Containers run as root
- No multi-stage builds
- Inefficient base images

### Running as Non-Root User

To run shared volumes as a non-root is challenging because:

1. Docker volumes are created with root ownership by default
2. Non-root users can't write to root-owned directories
3. Both containers need matching user IDs for consistent permissions

The solution is an **init container** that fixes permissions before other services start.

### Updated Docker Compose Configuration

Replace `compose.yaml` with this production-ready version:

```yaml
services:
  init:
    image: busybox
    volumes:
      - html:/html
    command: chown -R 101:101 /html

  backend:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - html:/html
    user: "101:101"
    depends_on:
      init:
        condition: service_completed_successfully

  frontend:
    image: nginxinc/nginx-unprivileged:1.29-alpine-perl
    ports:
      - "8080:8080"
    volumes:
      - html:/usr/share/nginx/html:ro
    depends_on:
      - backend
volumes:
  html:
```

#### Key improvements

**Init container**:
- Runs a lightweight BusyBox image
- Command `chown -R 101:101 /html` to change ownership to a non-root user.
- Runs before the other two services start

**Backend service**:
- Explicitly runs as user `101:101`
- Depends on the **init container** completing successfully
- Uses `condition: service_completed_successfully` to enforce startup order

**Frontend service**:
- Uses Alpine-based Nginx (smaller image)
- Nginx unprivileged image runs as user `101` by default
- Depends on the **backend service** being available

### Production-Ready Dockerfile

Update the `Dockerfile` with a multi-stage build:

```dockerfile
# Builder stage-1
FROM ubuntu:24.04 AS builder

WORKDIR /build
COPY fetch-joke .
RUN chmod +x fetch-joke

# Runtime stage-2
FROM alpine:3.23

# update and install Alpine dependencies
RUN apk add --no-cache curl ca-certificates bash

# Create non-root user
RUN addgroup -g 101 appgroup && \
    adduser -D -u 101 -G appgroup appuser

# Set working directory
WORKDIR /app

# Copy script from the builder stage
COPY --from=builder --chown=appuser:appgroup /build/fetch-joke .

# Switch to non-root user
USER appuser

CMD ["./fetch-joke"]
```

#### Key Components

**Multi-stage Build**:
- **Stage 1 (Builder)**: Ubuntu environment prepares the script with executable permissions
- **Stage 2 (Runtime)**: Minimal Alpine base contains only what's needed to run

**Dependencies**:
- `curl`: Make HTTP requests to the external API
- `ca-certificates`: Enable HTTPS connections
- `bash`: Alpine uses `ash` by default, but the script needs Bash
- `--no-cache`: Skips storing the package index to save space

**Security**:
```dockerfile
RUN addgroup -g 101 appgroup && \
    adduser -D -u 101 -G appgroup appuser
```
- Creates a group (GID 101) and user (UID 101)
- `-D`: No password (system user, not interactive)
- `-G`: Sets primary group membership
- Follows the **principle of least privilege**, never run containers as root

**Best Practices**:
1. Multi-stage build separates build/runtime concerns
2. Alpine base keeps image small (4MB vs 28MB Ubuntu)
3. Non-root user (UID 101) limits security exposure
4. Minimal dependencies reduce attack surface
5. `--chown` flag sets proper file ownership during copy

---

## Deploying Application

Run Docker Compose to rebuild and start the application with the security improvements:

```bash
docker compose down -v
docker compose up -d --build
```
- `-v`: Remove the Docker volume

Verify non-root execution:
```bash
docker compose exec frontend whoami
# Output: nginx

docker compose exec backend whoami
# Output: appuser
```

Both containers now run as non-root users with appropriate permissions on the shared volume.

### Understanding the Network and Volume Setup

When you run `docker compose up`, several things happen automatically:

1. **Network creation**: Docker Compose creates a dedicated network for the application, allowing containers to discover each other by the service name
2. **Volume creation**: The shared `html` storage is created
3. **Container startup**: Containers are started in dependency order
4. **Health monitoring**: Docker Compose monitors container health

---

## When to Use Docker Compose

Docker Compose is great for:

- **Local development**: Define your entire dev environment in one file
- **Simple deployments**: Single-host applications
- **Testing**: Spin up complete test environments quickly
- **Proof of concepts**: Rapid prototyping

Docker Compose has limitations:

- **Single host only**: Cannot distribute containers across multiple servers
- **No auto-restart**: Failed containers on remote hosts won't restart automatically
- **Zero-downtime updates**: Require taking down the entire application
- **Secret management**: Limited built-in secret handling
- **Load balancing**: No native support for distributing traffic

---

## The Bridge to Kubernetes

The concepts learned with Docker Compose translate directly to Kubernetes:

- **Services** become Kubernetes Services and Deployments
- **Volumes** become PersistentVolumes and PersistentVolumeClaims
- **Networks** become Kubernetes networking with DNS service discovery

Kubernetes addresses Docker Compose's limitations by providing:

- Multi-host orchestration
- Self-healing (automatic container restarts)
- Rolling updates with zero downtime
- Robust secrets management
- Built-in load balancing and service discovery

---

## Conclusion

I've built a complete multi-container application from scratch, progressing from a basic proof of concept to a production-ready deployment with robust security practices. Along the way, I've learned:

- How Docker Compose orchestrates multiple services through a single YAML file
- How containers communicate through shared volumes without direct networking
- How to implement security best practices with non-root users and multi-stage builds
- How init containers solve permission challenges in shared storage scenarios

> More importantly, I've followed a real-world development process: build it, make it work, then secure it. This pattern repeats throughout DevOps work.

---

## Definitions

**Docker Compose**: An orchestration tool for defining and running multi-container applications using a single YAML configuration file.

**Service**: A container definition in Docker Compose, including its image, volumes, ports, and dependencies. Each service can run multiple container instances.

**`compose.yaml`**: The Docker Compose configuration file that defines services, networks, and volumes.

**Volume**: A Docker-managed storage location that persists data and can be shared between containers.

**Multi-Container Application**: An application composed of multiple containers that work together, each handling a specific function (e.g., frontend, backend, database).

**Init Container**: A container that runs before other services start, typically used for setup tasks like fixing permissions or initializing data.

**Port Mapping**: Binding a container's internal port to a host port, formatted as `host_port:container_port` (e.g., `8080:80`).

**SSH Tunnel (Port Forwarding)**: A secure connection that forwards network traffic from a local port through an SSH connection to a remote host, enabling access to services running on remote servers.

**Orchestration**: The automated management of multiple containers, including deployment, scaling, networking, and lifecycle management.

**Build Context**: The directory containing files sent to the Docker daemon during image build, specified by (`.`) in the compose files.

**Detached Mode**: Running containers in the background (using `-d` flag), allowing the terminal to remain available for other commands.

**UID/GID**: User ID and Group ID numbers in Linux systems. Matching these across containers enables proper file permissions on shared volumes.

---
