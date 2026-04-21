# KIND Installation & Setup Guide

A step-by-step guide for installing and running **KIND** (Kubernetes IN Docker) — a tool for running local Kubernetes clusters using Docker containers as nodes. KIND is ideal for local development, testing, and CI/CD pipelines.

---

## ⚠️ Prerequisites

Before proceeding with KIND, ensure the following steps have been completed:

### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required before downloading KIND. `curl` is used in the next section to download the KIND binary.

### 3. Install & Configure Docker

```bash
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable --now docker
```

KIND runs each Kubernetes node as a Docker container, so Docker must be installed and running before any KIND cluster can be created. Running `sudo systemctl enable --now docker` again after `newgrp docker` ensures the daemon is fully active after the group change.

---

## 1. Download the KIND Binary

```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
```

This command is split into two parts separated by `&&`:

- **`[ $(uname -m) = x86_64 ]`** — a conditional check that verifies the system architecture is 64-bit (`x86_64`) before proceeding. If the architecture doesn't match, the download is skipped, preventing an incompatible binary from being downloaded.
- **`curl -Lo ./kind ...`** — downloads KIND version `v0.31.0` for 64-bit Linux from the official releases page and saves it as `./kind` in the current directory.

| Flag | Purpose |
|---|---|
| `-L` | Follows HTTP redirects |
| `-o ./kind` | Saves the output to a file named `kind` in the current directory |

> ℹ️ To install a different version, replace `v0.31.0` in the URL with your desired version. Check available releases at [https://github.com/kubernetes-sigs/kind/releases](https://github.com/kubernetes-sigs/kind/releases).

---

## 2. Make the Binary Executable

```bash
chmod +x ./kind
```

Grants execute permission to the downloaded `kind` binary. By default, downloaded files are not executable. Without this step, running `./kind` would result in a "Permission denied" error.

| Permission | Meaning |
|---|---|
| `+x` | Adds execute permission for the owner, group, and others |

---

## 3. Move KIND to System PATH

```bash
sudo mv ./kind /usr/local/bin/kind
```

Moves the `kind` binary from the current directory to `/usr/local/bin/`, making it available as a system-wide command accessible from any directory. After this step, the `./kind` file in the current directory no longer exists — it has been relocated, not copied.

> ℹ️ Unlike the `install` command used in Docker and kubectl setup, `mv` is used here. The executable permission set in the previous step is preserved during the move.

---

## 4. Verify KIND Installation

```bash
kind version
```

Prints the installed KIND version to the terminal (e.g., `kind v0.31.0 go1.xx linux/amd64`). Confirms the binary was installed correctly and is accessible from the command line.

---

## 🔌 Post-Requirement: kubectl

> **KIND creates and manages clusters, but you need kubectl to interact with them.**

Once a KIND cluster is running, use **kubectl** to deploy workloads, inspect resources, and manage the cluster. KIND automatically updates your `~/.kube/config` file with the new cluster's connection details upon creation. Install kubectl before proceeding to cluster creation.

---

## 5. Create the Cluster Configuration File

Before creating a cluster, you must create a YAML configuration file that defines the cluster topology. Save the following as `cluster-config-file.yml` in your working directory:

```bash
vi cluster-config-file.yml
```

This opens the file in the **vi** text editor. Follow these steps:

1. Press `i` to enter **Insert mode** (you will see `-- INSERT --` at the bottom of the terminal)
2. Paste or type the configuration below
3. Press `Esc` to exit Insert mode
4. Type `:wq` and press `Enter` to **write (save) and quit**

> ℹ️ If you made a mistake and want to exit without saving, press `Esc` then type `:q!` and press `Enter`.

Paste the following configuration:

---

### Cluster Configuration File

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.32.2
  - role: worker
    image: kindest/node:v1.32.2
  - role: worker
    image: kindest/node:v1.32.2
  - role: worker
    image: kindest/node:v1.32.2
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

After saving, verify the file was created correctly:

```bash
cat cluster-config-file.yml
```

This prints the contents of the file to the terminal. Confirm the output matches the configuration above before proceeding to cluster creation.

This configuration defines a **4-node cluster** with one control plane and three worker nodes. Here's a breakdown of each field:

| Field | Purpose |
|---|---|
| `kind: Cluster` | Declares this as a KIND Cluster resource definition |
| `apiVersion: kind.x-k8s.io/v1alpha4` | Specifies the KIND API version for the config schema |
| `role: control-plane` | This node manages the cluster — runs the API server, scheduler, and controller manager |
| `role: worker` | These nodes run application workloads (Pods) |
| `image: kindest/node:v1.32.2` | The Docker image used for each node, pinning the Kubernetes version to `v1.32.2` |
| `extraPortMappings` | Maps ports from a node container to the host machine, allowing external access to services running inside the cluster |
| `containerPort` | The port exposed inside the KIND node container (`80` for HTTP, `443` for HTTPS) |
| `hostPort` | The corresponding port on your host machine that traffic is forwarded from |
| `protocol: TCP` | Network protocol for the port mapping (`TCP` or `UDP`) |

> ℹ️ `extraPortMappings` is only defined on the last worker node in this example. Port `80` (HTTP) and port `443` (HTTPS) are mapped for ingress traffic, forwarding requests from your host machine into the cluster.

> ⚠️ Only **one** node needs `extraPortMappings` for ingress — typically the node where an Ingress Controller (e.g., NGINX) will be scheduled.

This file is referenced in the next step by the `--config` flag.

---

## 6. Create a Cluster

```bash
kind create cluster --name=cluster --config=cluster-config-file.yml
```

Provisions a new Kubernetes cluster using KIND with a custom name and configuration file:

| Flag | Purpose |
|---|---|
| `--name=cluster` | Assigns a name to the cluster (default is `kind` if omitted). Useful for running multiple clusters side by side |
| `--config=cluster-config-file.yml` | Points to a YAML file defining the cluster topology — number of nodes, roles, port mappings, and Kubernetes version |

KIND will pull the required `kindest/node` Docker images and start each node as a container. The `~/.kube/config` file is automatically updated with the new cluster's credentials and context.

---

## 7. Delete a Cluster

```bash
kind delete cluster --name=clustername
```

Permanently removes the specified KIND cluster and all its associated Docker containers, networks, and volumes. Replace `clustername` with the name used during `kind create cluster`.

> ⚠️ This is a **destructive operation** — all workloads, deployments, and cluster state are permanently erased. The corresponding entry is also removed from `~/.kube/config`.

---

## ✅ Quick Reference

```bash
# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable --now docker

# Download KIND (x86_64 only)
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64

# Make executable and install
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version

# Create cluster config file
vi cluster-config-file.yml

# Verify config file contents
cat cluster-config-file.yml

# Create cluster with config
kind create cluster --name=cluster --config=cluster-config-file.yml

# Delete cluster
kind delete cluster --name=clustername
```
