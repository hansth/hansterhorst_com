---
title: Commands and Arguments
date: 2026-05-17
weight: 1
---
# Docker Commands and Arguments

In container environments, **commands** define the executable binary that runs when a container starts, while **arguments** provide additional data, parameters, or flags to that executable.

---

## Docker Instructions

In a Dockerfile, two instructions control what runs when a container starts.

- `ENTRYPOINT`: The executable that runs when the container starts.
- `CMD`: The default arguments passed to `ENTRYPOINT`. Used only if no arguments are provided at runtime.

---

## Override the CMD

When a Dockerfile defines only `CMD`, the entire instruction can be overridden by passing a command at runtime.

```dockerfile
FROM ubuntu
CMD ["sleep", "5"]
```

```bash
docker run ubuntu sleep 10      # overrides CMD entirely
```

---

## Combining ENTRYPOINT with CMD

When a fixed executable is needed with variable arguments, use `ENTRYPOINT` for the executable and `CMD` for the default argument.

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

Running the container without arguments runs `sleep 5` as defined in the Dockerfile.

```bash
docker run ubuntu
```

Passing an argument at runtime overrides `CMD`.

```bash
docker run ubuntu 10            # runs: sleep 10
```

To override the `ENTRYPOINT`, use the `--entrypoint` flag.

```bash
docker run --entrypoint=watch ubuntu 20
```
- `--entrypoint=watch`: Replaces the `ENTRYPOINT` with `watch`.
- `20`: Passed as the argument to `watch`.

---

## Override Behavior

| `--entrypoint` Set? | Arguments Passed? | Result                                         |
|---------------------|-------------------|------------------------------------------------|
| No                  | No                | `ENTRYPOINT` and `CMD` run as defined          |
| No                  | Yes               | `ENTRYPOINT` kept, `CMD` replaced by arguments |
| Yes                 | No                | `ENTRYPOINT` replaced, `CMD` discarded         |
| Yes                 | Yes               | Both `ENTRYPOINT` and `CMD` fully overridden   |

> **Note**: Passing `--entrypoint` without arguments discards the Dockerfile `CMD`. The original `CMD` does not remain in effect.

---

## Build and Run Reference

**Build an image**

```bash
docker build -t webapp-color:lite .
```
- `-t`: Tags the image.
- `.`: Path to the Dockerfile.

**Run a container**

```bash
docker run -p 8282:8080 --name=webapp-color webapp-color:latest
```
- `-p 8282:8080`: Maps host port `8282` to container port `8080`.
- `--name`: Sets the container name.

**Run a command inside a container**

```bash
docker run python:3.6 cat /etc/os-release
```
- `cat`: The command passed to the container.
- `/etc/os-release`: The argument.

---
