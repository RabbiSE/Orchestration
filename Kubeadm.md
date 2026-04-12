# Kubeadm Installation & Setup Guide

A step-by-step guide for bootstrapping a production-grade multi-node Kubernetes cluster using **kubeadm** on Ubuntu/Debian-based Linux systems. This guide sets up one **Master (Control Plane)** node and one or more **Worker** nodes.

---

## ⚠️ Prerequisites

Before proceeding with kubeadm, ensure the following steps have been completed:

### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required before setting up Kubernetes components. `curl` is used throughout this guide to fetch GPG keys and manifests.

---

## 🔒 Security Group Configuration

> **Before proceeding, ensure port `6443` is exposed in your cloud Security Group (e.g., AWS, GCP, Azure).**

Port `6443` is the **Kubernetes API server port**. Worker nodes must be able to reach the master node on this port in order to join the cluster and communicate with the control plane.

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| `6443` | TCP | Inbound | Kubernetes API server — required for worker nodes to join and communicate with the master |

> ⚠️ Without this port open, `kubeadm join` on worker nodes will fail to connect to the control plane.

---

## 🖥️ Execute on BOTH Master & Worker Nodes

The following steps must be completed on **every node** in the cluster — both the master and all worker nodes — before initializing or joining the cluster.

---

### 1. Disable Swap

```bash
sudo swapoff -a
```

Disables swap memory on the node. Kubernetes requires swap to be disabled because it relies on accurate memory resource accounting for scheduling decisions. If swap is active, the kubelet will refuse to start by default.

> ⚠️ This disables swap only for the current session. To make it permanent across reboots, also comment out any swap entries in `/etc/fstab`.

---

### 2. Load Necessary Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Two actions performed here:

**Write the module config file:**
`cat <<EOF ... EOF` writes the module names to `/etc/modules-load.d/k8s.conf`, ensuring they are automatically loaded on every reboot.

**Load the modules immediately:**

| Module | Purpose |
|---|---|
| `overlay` | Enables OverlayFS — the filesystem driver used by container runtimes (containerd) to layer container images efficiently |
| `br_netfilter` | Enables bridge network filtering — required so that iptables rules can process traffic crossing Kubernetes pod network bridges |

---

### 3. Set Sysctl Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
lsmod | grep br_netfilter
lsmod | grep overlay
```

Configures kernel networking parameters required for Kubernetes to route traffic correctly between pods and nodes:

| Parameter | Purpose |
|---|---|
| `net.bridge.bridge-nf-call-iptables = 1` | Ensures IPv4 traffic crossing network bridges is processed by iptables — required for pod-to-pod networking |
| `net.bridge.bridge-nf-call-ip6tables = 1` | Same as above for IPv6 traffic |
| `net.ipv4.ip_forward = 1` | Enables IP forwarding, allowing the node to route packets between network interfaces — essential for pod networking |

- **`sudo sysctl --system`** — applies all sysctl settings from config files immediately without a reboot
- **`lsmod | grep br_netfilter`** — verifies the `br_netfilter` module is loaded; should return a non-empty result
- **`lsmod | grep overlay`** — verifies the `overlay` module is loaded; should return a non-empty result

> ⚠️ If either `lsmod` check returns no output, the module did not load correctly. Re-run the corresponding `modprobe` command from Step 2.

---

### 4. Install Containerd

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

containerd config default | sed -e 's/SystemdCgroup = false/SystemdCgroup = true/' -e 's/sandbox_image = "registry.k8s.io\/pause:3.6"/sandbox_image = "registry.k8s.io\/pause:3.9"/' | sudo tee /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl status containerd
```

Installs **containerd** as the container runtime for Kubernetes. Unlike Docker, kubeadm clusters use containerd directly as the CRI (Container Runtime Interface). This step is broken into several parts:

**Install dependencies and add Docker's GPG key:**
- `ca-certificates` and `curl` are required to securely fetch the Docker repository GPG key
- `sudo install -m 0755 -d /etc/apt/keyrings` — creates the keyrings directory with correct permissions
- The `curl` command downloads Docker's official GPG key to `/etc/apt/keyrings/docker.asc`
- `chmod a+r` — makes the key file readable by all users so `apt` can verify package signatures

**Add Docker's apt repository:**
The `echo ... | sudo tee` command constructs and writes the Docker apt source entry to `/etc/apt/sources.list.d/docker.list`:

| Variable | Purpose |
|---|---|
| `$(dpkg --print-architecture)` | Dynamically inserts the system architecture (e.g., `amd64`) |
| `$(. /etc/os-release && echo "$VERSION_CODENAME")` | Dynamically inserts the Ubuntu codename (e.g., `jammy`, `focal`) |

**Install containerd:**
`containerd.io` is the production-grade containerd package from Docker's repository.

**Configure containerd for Kubernetes:**
The `containerd config default | sed ...` pipeline generates a default config and applies two critical modifications:

| Change | Purpose |
|---|---|
| `SystemdCgroup = false` → `true` | Enables systemd cgroup driver — required for kubelet and containerd to use the same cgroup manager |
| `pause:3.6` → `pause:3.9` | Updates the sandbox (pause) container image to a version compatible with Kubernetes v1.29+ |

- **`sudo systemctl restart containerd`** — applies the new configuration
- **`sudo systemctl status containerd`** — confirms the service is running correctly

---

### 5. Install Kubernetes Components

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Adds the official Kubernetes apt repository and installs the three core components:

| Package | Purpose |
|---|---|
| `kubelet` | The node agent that runs on every node — manages pods and communicates with the control plane |
| `kubeadm` | The cluster bootstrapping tool — used to initialize the master and join worker nodes |
| `kubectl` | The CLI tool for interacting with the cluster |

**Steps breakdown:**
- The `curl | gpg --dearmor` command fetches the Kubernetes GPG signing key and converts it to binary format, saving it to `/etc/apt/keyrings/kubernetes-apt-keyring.gpg`
- The `echo ... | tee` command adds the Kubernetes v1.29 stable apt repository to the sources list
- **`sudo apt-mark hold kubelet kubeadm kubectl`** — prevents these packages from being automatically upgraded by `apt`, ensuring cluster component versions stay in sync and avoid unintended breaking changes

> ℹ️ To install a different Kubernetes version, replace `v1.29` in both the `curl` URL and the `echo` repository line with your desired version (e.g., `v1.30`, `v1.31`).

---

## 👑 Execute ONLY on the Master Node

The following steps are performed **once on the master (control plane) node only**.

---

### 1. Initialize the Cluster

```bash
sudo kubeadm init
```

Bootstraps the Kubernetes control plane on the master node. This single command performs a sequence of operations including pre-flight checks, certificate generation, etcd setup, and control plane component deployment.

On success, kubeadm will print:
- A confirmation that the control plane is running
- The `kubeconfig` setup commands (see Step 2)
- The **`kubeadm join`** command with a token — **copy and save this immediately** as it is needed to join worker nodes (see Step 4)

> ℹ️ Additional flags such as `--pod-network-cidr=192.168.0.0/16` may be required depending on the network plugin you plan to install. Calico (used in Step 3) works without this flag by default.

---

### 2. Set Up Local kubeconfig

```bash
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
```

Configures `kubectl` on the master node to communicate with the newly created cluster:

| Command | Purpose |
|---|---|
| `mkdir -p "$HOME"/.kube` | Creates the `.kube` directory in the home folder if it doesn't exist |
| `sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config` | Copies the cluster admin credentials file generated by `kubeadm init` to the standard kubectl config location |
| `sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config` | Sets ownership of the config file to the current user so kubectl can read it without `sudo` |

> ℹ️ `$(id -u)` and `$(id -g)` resolve to the current user's UID and GID respectively, ensuring correct ownership regardless of the username.

---

### 3. Install a Network Plugin (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

Installs **Calico** as the Container Network Interface (CNI) plugin. Kubernetes does not provide pod-to-pod networking out of the box — a CNI plugin is required for pods across nodes to communicate.

Calico provides:
- Pod network routing between nodes
- Network policy enforcement (firewall rules between pods)
- Support for both overlay and direct routing modes

> ℹ️ Nodes will remain in a `NotReady` state until a CNI plugin is installed. After applying this manifest, allow a minute or two for Calico pods to become ready.

---

### 4. Generate Join Command

```bash
kubeadm token create --print-join-command
```

Generates a new bootstrap token and prints the complete `kubeadm join` command that worker nodes must run to join the cluster. The output will look similar to:

```
kubeadm join <private-ip-of-control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

> ⚠️ **Copy this output immediately.** You will need it in the Worker Node section below. Tokens expire after 24 hours by default — run this command again if the token has expired.

---

## 👷 Execute on ALL Worker Nodes

The following steps are performed on **each worker node** to join them to the cluster.

---

### 1. Perform Pre-flight Checks

```bash
sudo kubeadm reset pre-flight checks
```

Resets any previous kubeadm state and runs pre-flight validation checks on the worker node to ensure it is in a clean state and ready to join the cluster. This is a safety step that clears any leftover configuration from previous attempts.

---

### 2. Join the Cluster

Paste the join command copied from the master node's Step 4 output, with the following modifications:

1. Add `sudo` at the beginning
2. Append `--cri-socket "unix:///run/containerd/containerd.sock"` to explicitly specify the container runtime
3. Append `--v=5` at the end for verbose output to help diagnose any issues

The final command should follow this format:

```bash
sudo kubeadm join <private-ip-of-control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket "unix:///run/containerd/containerd.sock" --v=5
```

| Argument/Flag | Purpose |
|---|---|
| `<private-ip-of-control-plane>:6443` | The private IP address of the master node and the Kubernetes API server port |
| `--token <token>` | The bootstrap token generated by the master node to authenticate the join request |
| `--discovery-token-ca-cert-hash sha256:<hash>` | A hash of the master's CA certificate to verify the cluster identity and prevent man-in-the-middle attacks |
| `--cri-socket "unix:///run/containerd/containerd.sock"` | Explicitly tells kubeadm to use containerd as the container runtime interface |
| `--v=5` | Sets verbosity level to 5 — prints detailed logs useful for troubleshooting a failed join |

> ⚠️ Replace `<private-ip-of-control-plane>`, `<token>`, and `<hash>` with the actual values from the join command printed on the master node.

---

## ✅ Verify Cluster Connection

Run the following on the **Master Node** once all worker nodes have joined:

```bash
kubectl get nodes
```

Lists all nodes registered in the cluster along with their status. A healthy cluster will show all nodes as `Ready`:

```
NAME           STATUS   ROLES           AGE   VERSION
master-node    Ready    control-plane   5m    v1.29.x
worker-node-1  Ready    <none>          2m    v1.29.x
worker-node-2  Ready    <none>          2m    v1.29.x
```

> ℹ️ Nodes may show as `NotReady` for a minute or two after joining while the CNI plugin and kubelet finish initializing. Re-run `kubectl get nodes` after a short wait if nodes are not yet `Ready`.

---

## ✅ Quick Reference

```bash
# ---- BOTH MASTER & WORKER NODES ----

# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree

# Disable swap
sudo swapoff -a

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
lsmod | grep br_netfilter
lsmod | grep overlay

# Install containerd
sudo apt-get update && sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y containerd.io
containerd config default | sed -e 's/SystemdCgroup = false/SystemdCgroup = true/' -e 's/sandbox_image = "registry.k8s.io\/pause:3.6"/sandbox_image = "registry.k8s.io\/pause:3.9"/' | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl status containerd

# Install Kubernetes components
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# ---- MASTER NODE ONLY ----

# Initialize cluster
sudo kubeadm init

# Set up kubeconfig
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Install Calico CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Generate join command (copy output for worker nodes)
kubeadm token create --print-join-command

# ---- WORKER NODES ONLY ----

# Reset and join
sudo kubeadm reset pre-flight checks
sudo kubeadm join <private-ip-of-control-plane>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket "unix:///run/containerd/containerd.sock" --v=5

# ---- VERIFY ON MASTER NODE ----
kubectl get nodes
```
