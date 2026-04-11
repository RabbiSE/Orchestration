# Minikube Installation & Setup Guide

A step-by-step guide for installing and running **Minikube** — a local Kubernetes environment — on a Linux system using Docker as the underlying driver.

---

## ⚠️ Prerequisites

Before proceeding with Minikube, you must have completed the following steps from the **Docker Installation & Setup Guide**:

### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required for downloading and setting up Minikube and Docker. Notably, `curl` is used in the very next section to download the Minikube binary, and `apt-transport-https` is required to allow `apt` to fetch packages over HTTPS.

### 3. Install Docker Engine

```bash
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

Minikube uses Docker as its underlying driver to run the Kubernetes node. Docker must be installed, the service must be running, and your user must have Docker group access before `minikube start` will work.

> 📄 For detailed explanations of each Docker command above, refer to the [Docker Installation & Setup Guide](./Docker.md).

---

## 1. Download the Minikube Binary

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
```

Downloads the latest Minikube binary for 64-bit Linux directly from the official GitHub releases page:

| Flag | Purpose |
|---|---|
| `-L` | Follows HTTP redirects (GitHub releases use redirects) |
| `-O` | Saves the file with its original filename (`minikube-linux-amd64`) |

The file is saved to your current working directory and is not yet executable or installed — that happens in the next step.

---

## 2. Install Minikube & Clean Up

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

Two actions chained together with `&&` (the second only runs if the first succeeds):

- **`sudo install minikube-linux-amd64 /usr/local/bin/minikube`** — copies the downloaded binary to `/usr/local/bin/`, sets the correct ownership and executable permissions in one step. This makes `minikube` available system-wide as a CLI command.
- **`rm minikube-linux-amd64`** — removes the leftover downloaded file from the current directory, keeping things tidy.

> ℹ️ `/usr/local/bin/` is a standard location for user-installed binaries and is included in `$PATH` by default on most Linux systems.

---

## 3. Verify Minikube Installation

```bash
minikube version
```

Prints the installed Minikube version to the terminal (e.g., `minikube version: v1.x.x`). A quick sanity check to confirm the binary was installed correctly and is accessible from the command line.

---

## 4. Start the Minikube Cluster

```bash
minikube start --driver=docker --vm=true
```

Bootstraps a single-node local Kubernetes cluster. On first run, Minikube will download the required Kubernetes component images automatically.

| Flag | Purpose |
|---|---|
| `--driver=docker` | Uses Docker as the container runtime/driver to run the Kubernetes node |
| `--vm=true` | Runs the cluster inside a VM-like isolated environment within Docker, providing better network isolation |

> ⚠️ Requires Docker to be running and your user to have Docker group access (see the Docker setup guide). Minikube will allocate CPU and memory from your host machine for the cluster.

---

## 5. Enable the Ingress Addon

```bash
minikube addons enable ingress
```

Enables the **NGINX Ingress Controller** addon inside the Minikube cluster. Ingress allows you to route external HTTP/HTTPS traffic to services running inside Kubernetes based on rules (hostnames, paths, etc.) — essentially acting as a reverse proxy and load balancer for your cluster.

> ℹ️ Minikube ships with a library of optional addons (e.g., `dashboard`, `metrics-server`, `registry`). Run `minikube addons list` to see all available options.

---

## 6. Check Cluster Status

```bash
minikube status
```

Displays the current state of the Minikube cluster and its core components. A healthy output will show:

```
minikube: Running
cluster: Running
kubectl: Correctly Configured
```

Use this to confirm the cluster is up and reachable, or to diagnose why a cluster may not be responding as expected.

---

## 7. Stop the Cluster

```bash
minikube stop
```

Gracefully shuts down the running Minikube cluster and the underlying Docker container/VM. All Kubernetes resources and cluster state are **preserved** — the next `minikube start` will resume where you left off. Use this when you're done working and want to free up system resources (CPU, memory) without losing your cluster configuration.

---

## 8. Delete the Cluster

```bash
minikube delete
```

Completely removes the Minikube cluster, including all associated containers, volumes, and configuration data. This is a **destructive operation** — all workloads, deployments, and cluster state are permanently erased.

> ⚠️ Use this when you want a clean slate or need to free up disk space. A subsequent `minikube start` will provision a brand new cluster from scratch.

---

## ✅ Quick Reference

```bash
# Install Minikube
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# Verify
minikube version

# Start cluster with Docker driver
minikube start --driver=docker --vm=true

# Enable Ingress addon
minikube addons enable ingress

# Check status
minikube status

# Stop cluster (preserves state)
minikube stop

# Delete cluster (permanent)
minikube delete
```
