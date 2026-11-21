# Podman Networking

> Notes and practical commands from my Red Hat **DO188 (Containers with Podman)** training.

![Status](https://img.shields.io/badge/status-learning%20in%20progress-blue)
![Podman](https://img.shields.io/badge/podman-5.x-purple)

## Table of Contents

- [1. Introduction to Container Networking](#1-introduction-to-container-networking)
- [2. Podman Network Management Commands](#2-podman-network-management-commands)
- [3. Practical Examples](#3-practical-examples)
- [4. Network Isolation](#4-network-isolation)
- [5. Container-to-Container Communication](#5-container-to-container-communication)
- [6. Important Concepts](#6-important-concepts)
- [7. Troubleshooting: Short Image Names](#7-troubleshooting-short-image-names)
- [8. Official Documentation](#8-official-documentation)
- [9. Practice Checklist](#9-practice-checklist)

---

## 1. Introduction to Container Networking

Container networking allows containers to communicate with each other and with external networks.

For example, consider a web application composed of three containers:

| Container | Role |
|:---|:---|
| UI | Frontend |
| API | Backend |
| Database | PostgreSQL |

### Network Isolation Example

Instead of allowing every container to communicate with every other container, we can create separate networks:

- `ui-network`: connects the UI and API containers.
- `api-network`: connects the API and Database containers.

**Benefits:**

- Restricts unnecessary communication between containers.
- Improves network security.
- Separates application components.
- Controls which containers can communicate with each other.

---

## 2. Podman Network Management Commands

Podman provides the `podman network` subcommand to manage container networks.

| Command | Description |
|:---|:---|
| `podman network create` | Creates a new network. |
| `podman network ls` | Lists existing networks. |
| `podman network inspect` | Displays detailed network configuration. |
| `podman network rm` | Removes a network. |
| `podman network prune` | Removes unused networks. |
| `podman network connect` | Connects an existing container to a network. |
| `podman network disconnect` | Disconnects a container from a network. |

---

## 3. Practical Examples

### 3.1 Create a Network

Creates a new network named `example-net`.

```bash
podman network create example-net
```

### 3.2 List Networks

Displays all existing Podman networks.

```bash
podman network ls
```

### 3.3 Inspect a Network

Displays detailed information about a network, including its configuration.

```bash
podman network inspect example-net
```

### 3.4 Start a Container on a Network

Use the `--network` (or `--net`) option to attach a container to a specific network when creating it.

```bash
podman run -d \
  --name my-container \
  --network example-net \
  docker.io/library/nginx:latest
```

> **Note:** Use the fully qualified image name (`docker.io/library/nginx:latest`) rather than `nginx`. See [Troubleshooting](#7-troubleshooting-short-image-names).

### 3.5 Connect an Existing Container

Connects an already running container to an existing network.

```bash
podman network connect example-net my-container
```

### 3.6 Disconnect a Container

Disconnects a container from a network.

```bash
podman network disconnect example-net my-container
```

### 3.7 Remove a Network

Removes an existing network, provided it is not in use.

```bash
podman network rm example-net
```

### 3.8 Remove Unused Networks

Removes networks that are not currently used by containers.

```bash
podman network prune
```

---

## 4. Network Isolation

Starting with Podman 4.2.0, the `isolate` option is supported by the default bridge network driver.

It allows you to isolate networks from other networks that also have isolation enabled.

### Create an Isolated Network

```bash
podman network create -o isolate=true ui-network
```

Create another isolated network:

```bash
podman network create -o isolate=true api-network
```

> **Important:** Containers connected to different isolated networks cannot communicate directly through those isolated networks.

---

## 5. Container-to-Container Communication

Containers connected to the same DNS-enabled Podman network can communicate using their container names instead of IP addresses.

### Step 1: Create the Network and Start an Nginx Container

```bash
podman network create test-net

podman run -d \
  --name nginx-container \
  --network test-net \
  docker.io/library/nginx:latest
```

### Step 2: Access Nginx from Another Container

Run a second container on the same network and query Nginx by name:

```bash
podman run --rm \
  --network test-net \
  docker.io/curlimages/curl:latest \
  http://nginx-container:80
```

**Explanation:**

- `nginx-container`: container name, used as the hostname.
- `80`: HTTP port used by Nginx.
- Podman's DNS resolves the container name to its current IP address.

This makes communication easier because you do not need to manually identify container IP addresses.

---

## 6. Important Concepts

| Concept | Explanation |
|:---|:---|
| **Rootful containers** | Containers running with root privileges. |
| **Rootless containers** | Containers running without root privileges. |
| **Bridge network** | A virtual network that connects containers. |
| **DNS** | Allows containers to communicate using container names. |
| **Pasta** | The default rootless networking backend starting with Podman 5. |

### Additional Notes

- Rootful containers use the configured Podman network infrastructure.
- The default Podman network has DNS disabled by default.
- Rootless containers do not automatically attach to the default Podman network.
- To enable communication between rootless containers, create a network and attach the containers to it.
- Starting with Podman 5, `pasta` replaced `slirp4netns` as the default rootless networking backend.

### Important: Pasta Networking

When a container is created without specifying `--network`, Podman uses the default rootless networking mode (`pasta`).

A container using `pasta` cannot be connected directly to a Podman bridge network using `podman network connect`.

To attach it to a bridge network, recreate the container with the desired `--network` option.

---

## 7. Troubleshooting: Short Image Names

### Problem

Podman returns:

```text
Error: short-name "nginx" did not resolve to an alias
and no unqualified-search registries are defined
```

### Cause

Podman cannot determine which container registry should be used to pull the image.

### Solution

Use the fully qualified image name:

```bash
podman run -d \
  --name my-container \
  --network example-net \
  docker.io/library/nginx:latest
```

### Key Takeaway

Prefer fully qualified image names to avoid short-name resolution issues.

| Type | Example |
|:---|:---|
| Short name | `nginx` |
| Fully qualified | `docker.io/library/nginx:latest` |

---

## 8. Official Documentation

- [Podman Networking Documentation](https://docs.podman.io/en/v5.8.2/markdown/podman-network.1.html)
- [Podman Network Create](https://docs.podman.io/en/v5.8.2/markdown/podman-network-create.1.html)
- [Basic Networking Guide for Podman](https://github.com/podman-container-tools/podman/blob/v5.8.2/docs/tutorials/basic_networking.md)

---

## 9. Practice Checklist

Use this checklist to track your hands-on learning.

- [ ] Create a Podman network.
- [ ] List and inspect existing networks.
- [ ] Start two containers on the same network.
- [ ] Test container-to-container communication.
- [ ] Connect an existing container to a network.
- [ ] Disconnect a container from a network.
- [ ] Create isolated networks.
- [ ] Understand rootful vs. rootless networking.
- [ ] Test DNS-based communication using container names.
- [ ] Inspect and remove unused networks.

---

**Status:** Learning in progress 🚀
