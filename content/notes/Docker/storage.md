---
title: Docker Storage
date: 2026-03-19
---
# Docker Storage

**Docker storage** uses a layered architecture separating **read-only image data** from the **container's writable state**. Understanding file systems, layers, copy-on-write, volumes, and storage drivers is essential for confident, effective container management with Kubernetes.

---

## File System

When you install Docker on a system, it creates a file structure at `/var/lib/docker`. In this, you will find directories such as `buildkit`, `containers`, `image`, and `volumes`. This is where Docker stores all its data by default.

```
/var/lib/docker
│
├── buildkit
├── container
├── image
├── ...
└── volumes
```

Files related to containers are stored under the `containers` directory. Files related to images are stored under the `image` directories. Any volumes created by Docker containers are created under the `volumes` directory.

---

## Layered Architecture

Docker builds images in a **layered architecture**. When you build the image, each instruction in a Dockerfile creates a new layer containing only the changes from the previous layer.

```bash
docker build -t pip-web-app:1.0.0 ./Dockerfile
```

Dockerfile to build the Docker image
```dockerfile
# Layer 1
FROM Ubuntu
# Layer 2
RUN apt-get update && apt-get -y install python
# Layer 3
RUN pip install flask flask-mysql
# Layer 4
COPY . /opt/source-code
# Layer 5
ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run
```

Layer structure of container image
```
┌────────────────────────────────────────────┐
│  Image Read-Only Layer                     │
│ ┌─────────────────────────────────┐        │
│ │  Layer 5. Entrypoint flask CMD  │        │
│ └─────────────────────────────────┘        │
│ ┌─────────────────────────────────┐        │
│ │  Layer 4. Source Code.          │        │
│ └─────────────────────────────────┘        │
│ ┌─────────────────────────────────┐        │
│ │  Layer 3. Changes pip packages  │        │
│ └─────────────────────────────────┘        │
│ ┌─────────────────────────────────┐        │
│ │  Layer 2. Changes apt packages  │        │
│ └─────────────────────────────────┘        │
│ ┌─────────────────────────────────┐        │
│ │  Layer 1. Base Ubuntu Layer     │        │
│ └─────────────────────────────────┘        │
└────────────────────────────────────────────┘
```

- Layer 1 is the base Ubuntu distribution.
- Layer 2 installs apt packages.
- Layer 3 installs pip packages.
- Layer 4 copies the source code.
- Layer 5 sets the entry point.

Because each layer only stores the changes between the two states (delta) from the previous one, the size reflects only what changed.

### Layer Caching

When a second application shares the same Base image with the Python and Flask dependencies but uses different source code. When building that second image, Docker reuses the first three layers from cache and only creates the last two layers with the new source and entrypoint. This makes the build process faster and saves disk space.

The same applies when updating application code. Docker reuses all unchanged layers from cache and only rebuilds from the modified layer onward.

### Runtime Layer

Once a build is complete, the **image layers are read-only**. They cannot be modified without initiating a new build.

```bash
docker run pip-web-app:1.0.0
```

When you run the container, Docker creates a new **writable layer** on top of the image layers.

```
┌────────────────────────────────────────────┐
│  Container Read/Write Layer                │
│ ┌─────────────────────────────────┐        │
│ │  Layer 6. Container Layer       │        │
│ └─────────────────────────────────┘        │
└────────────────────────────────────────────┘
```

This writable container layer stores log files, temporary files, and any other data written at runtime. Its lifetime is tied to the container: when the container is destroyed, this layer and everything stored in it are destroyed as well. The underlying image layers are shared by all containers created from the same image.

---

## Copy-on-Write

The image layers are read-only, but that doesn't prevent you from modifying files within a running container.

When you log into a container and want to edit `app.py` to test a change. The file lives in the image layer and is shared across all containers using that image. Before saving the modification, Docker automatically copies the file into the writable container layer. All modifications are applied to that copy. The original file in the image layer remains untouched.

This mechanism is called **copy-on-write**. The image itself is never modified at runtime. It stays consistent until you rebuild it with `docker build`.

When the container is removed, the copied file in the container layer is also removed along with any other runtime changes.

---

## Volumes

Containers are ephemeral; when a container is destroyed, the writable layer and its data are lost. To persist data beyond the life of a container, you create and use a volume.

Volumes are persistent data stores for containers, created and managed by Docker.

### Volume Mounting

Create a volume with the `docker volume create` command:

```bash
docker volume create data_volume
```

This creates a directory at `/var/lib/docker/volumes/data_volume`. Mount it when running a container using the `-v` flag:

```bash
docker run -v data_volume:/var/lib/mysql mysql
```
- `data_volume`: The mounted volume directory on the Docker host
- `:/var/lib/mysql`:  Directory inside the container.

All data written by the MySQL database is stored in the volume directory on the Docker host. Even if the container is destroyed, the data remains. If the named volume does not exist when `docker run` is called, Docker creates it automatically.

### Bind Mounting

Alternatively, if you prefer using an external directory on the host rather than the Docker-managed volumes directory, you can use bind mounts.

```bash
docker run -v /data/mysql:/var/lib/mysql mysql
```
This mounts the host directory `/data/mysql` to the container directory.

This is called bind mounting. The difference between the two types is:

- **Volume mount** uses a volume from the Docker filesystem `/var/lib/docker/volumes/`
- **Bind mount** uses any directory from the host file system

## Recommended Using the `--mount` Flag

Using `-v` is the older approach. The newer `--mount` flag provides a clearer and more meaningful syntax.

```bash
docker run \
  --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql
```

Each parameter is specified as a `key=value` pair using `type`, `source`, and `target`.

---

## Storage Drivers

The **layered architecture**, **writable container layer**, and **copy-on-write** mechanism are all managed by Docker's storage drivers. Storage drivers are responsible for:

- Maintaining the layered image structure
- Creating the writable container layer
- Moving files across layers to implement copy-on-write

Common storage drivers include AUFS, Btrfs, ZFS, Device Mapper, Overlay, and Overlay2. The choice of driver depends on the underlying operating system. On Ubuntu, the default is AUFS. On Fedora or CentOS, Device Mapper is typically used. Docker selects the best available driver automatically based on the OS. Different drivers also offer different performance and stability characteristics, so the right choice may depend on your workload.

### Volume Driver Plugins

Volumes are not handled by storage drivers. They are handled by volume driver plugins. The default plugin is `local`, which creates volumes on the Docker host under `/var/lib/docker/volumes/`.

Third-party volume driver plugins extend this to external storage providers, including:

- Azure File Storage
- DigitalOcean Block Storage
- Google Compute Persistent Disks
- Amazon EFS
- NetApp
- VMware vSphere Storage
- Portworx

The Rex-Ray storage driver, for example, can provision storage on AWS EBS, S3, EMC arrays, Google Persistent Disk, and OpenStack Cinder. To use it, specify the driver when running a container:

```bash
docker run --volume-driver rexray/ebs \
  --mount src=ebs-vol,target=/data \
  <container_image>
```

This attaches an EBS volume from AWS. When the container exits, the data remains safely stored in the cloud.

---
