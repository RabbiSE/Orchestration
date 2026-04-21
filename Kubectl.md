# kubectl Installation & Setup Guide

A step-by-step guide for installing **kubectl** — the official Kubernetes command-line tool — on a Linux system. kubectl allows you to deploy applications, inspect and manage cluster resources, and view logs across any Kubernetes cluster.

---

## ⚠️ Prerequisites

Before proceeding with kubectl, ensure the following steps have been completed:

### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required before downloading kubectl. Notably, `curl` is used twice in the kubectl download command — once to resolve the latest stable version and once to download the binary itself.

---

## 🔌 Kubernetes Cluster Requirement

> **kubectl is a client tool only — it does not run or include a Kubernetes cluster.**

kubectl must be connected to a running Kubernetes cluster to do anything meaningful. Once installed, you should pair it with one of the following:

| Cluster Tool | Best For |
|---|---|
| **Minikube** | Local single-node development & testing |
| **KIND** (Kubernetes IN Docker) | Lightweight local clusters, CI/CD pipelines |
| **Kubeadm** | Production-grade multi-node cluster bootstrapping |

kubectl automatically reads cluster connection details (API server address, credentials) from a configuration file located at `~/.kube/config`, which is generated when you set up any of the cluster tools above.

---

## 1. Download the kubectl Binary

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Downloads the latest stable kubectl binary for 64-bit Linux. This command is composed of two nested `curl` calls:

- **Inner command** `$(curl -L -s https://dl.k8s.io/release/stable.txt)` — fetches the latest stable Kubernetes version string (e.g., `v1.30.0`) from the official Kubernetes release API
- **Outer command** `curl -LO "https://dl.k8s.io/release/.../kubectl"` — uses that version string to construct the full download URL and fetch the binary

| Flag | Purpose |
|---|---|
| `-L` | Follows HTTP redirects |
| `-O` | Saves the file with its original filename (`kubectl`) |
| `-s` | Silent mode — suppresses progress output for the inner curl |

The binary is saved to your current working directory and is not yet executable or installed — that happens in the next step.

---

## 2. Install kubectl System-Wide

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Installs the downloaded binary to `/usr/local/bin/` with secure, correct permissions in a single command:

| Flag | Purpose |
|---|---|
| `-o root` | Sets the file **owner** to `root` |
| `-g root` | Sets the file **group** to `root` |
| `-m 0755` | Sets file permissions to `rwxr-xr-x` — owner can read/write/execute, everyone else can read/execute |

> ℹ️ `/usr/local/bin/` is included in `$PATH` by default on most Linux systems, making `kubectl` available as a command from any directory. After installation, the downloaded `kubectl` file in your current directory can be safely removed with `rm kubectl`.

---

## 3. Verify kubectl Installation

```bash
kubectl version --client
```

Prints the installed kubectl client version (e.g., `Client Version: v1.30.0`). The `--client` flag ensures the command only checks the local binary version without attempting to contact a cluster — useful at this stage since no cluster may be configured yet.

> ℹ️ Once connected to a cluster, running `kubectl version` (without `--client`) will display both the client version and the Kubernetes server version of the connected cluster.

---

## ✅ Quick Reference

```bash
# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree

# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```
