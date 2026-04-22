# Amazon EKS Installation & Setup Guide

A step-by-step guide for provisioning and connecting to an **Amazon EKS (Elastic Kubernetes Service)** cluster on AWS. EKS is a fully managed Kubernetes service that removes the need to install and operate your own control plane.

---

## ⚠️ Prerequisites

Before proceeding with EKS, ensure the following steps have been completed:

### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required before downloading EKS tooling. `curl` is used throughout this guide to download binaries and scripts, and `unzip` is required to extract the AWS CLI archive.

---

## 🔒 IAM Role Configuration

> **Before installing any tooling, an IAM role must be created and attached to your EC2 instance.**

EKS management tools (`aws`, `eksctl`) need AWS permissions to create and manage cloud resources. Rather than hardcoding credentials, the recommended approach is to attach an **IAM role** directly to the EC2 instance — the AWS CLI and eksctl will automatically use the instance's role credentials.

### Steps to create and attach the IAM role:

1. Open the **AWS Console** and navigate to **IAM → Roles → Create Role**
2. Select **Trusted entity type: AWS Service** and use case: **EC2**
3. Attach the **`AdministratorAccess`** policy
4. Name the role (e.g., `eks-admin-role`) and create it
5. Navigate to **EC2 → Instances**, select your instance
6. Click **Actions → Security → Modify IAM Role**
7. Select the role you just created and click **Update IAM Role**

> ⚠️ `AdministratorAccess` grants full AWS permissions and is appropriate for lab and learning environments. For production use, scope the permissions down to only what EKS and eksctl require.

---

## 1. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Downloads and installs the latest stable kubectl binary. This command is composed of two parts:

**Download:**
The outer `curl -LO` uses the result of the inner `curl` to dynamically construct the download URL for the latest stable version, saving it to the current directory.

| Flag | Purpose |
|---|---|
| `-L` | Follows HTTP redirects |
| `-O` | Saves with the original filename (`kubectl`) |
| `-s` | Silent mode for the inner curl — suppresses progress output |

**Install:**

| Flag | Purpose |
|---|---|
| `-o root` | Sets the file owner to `root` |
| `-g root` | Sets the file group to `root` |
| `-m 0755` | Sets permissions to `rwxr-xr-x` — owner can read/write/execute, everyone else can read/execute |

> ℹ️ After installation, the downloaded `kubectl` file in the current directory can be safely removed with `rm kubectl`.

---

## 2. Install the AWS CLI

```bash
sudo apt-get -y install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Installs the **AWS CLI v2** — the command-line tool for interacting with AWS services. Required by eksctl to authenticate and make AWS API calls.

| Command | Purpose |
|---|---|
| `sudo apt-get -y install unzip` | Installs the `unzip` utility needed to extract the downloaded archive |
| `curl ... -o "awscliv2.zip"` | Downloads the AWS CLI v2 installer zip from the official AWS endpoint |
| `unzip awscliv2.zip` | Extracts the installer into an `aws/` directory in the current folder |
| `sudo ./aws/install` | Runs the installer, placing the `aws` binary in `/usr/local/bin/` |

> ℹ️ The AWS CLI will automatically use the IAM role attached to the EC2 instance for authentication — no `aws configure` is needed when running on an EC2 instance with an attached role.

---

## 3. Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

Installs **eksctl** — the official CLI tool for creating and managing EKS clusters. It abstracts the complexity of setting up VPCs, subnets, node groups, and IAM roles into a single command.

**Download and extract in one step:**

| Part | Purpose |
|---|---|
| `--silent` | Suppresses progress output |
| `--location` | Follows HTTP redirects |
| `$(uname -s)` | Dynamically inserts the OS name (e.g., `Linux`) to construct the correct download URL |
| `tar xz -C /tmp` | Extracts the `.tar.gz` archive directly into `/tmp` without saving the compressed file to disk |

**Install:**
`sudo mv /tmp/eksctl /usr/local/bin` — moves the extracted binary to `/usr/local/bin/`, making it available system-wide as a CLI command.

> ℹ️ Unlike kubectl and the AWS CLI, eksctl is installed via a direct pipe into `tar` — no intermediate file is saved to disk.

---

## 4. Verify Installations

```bash
kubectl version --client
aws --version
eksctl version
```

Confirms all three tools are installed correctly and accessible from the command line:

| Command | Expected Output |
|---|---|
| `kubectl version --client` | Prints kubectl client version (e.g., `Client Version: v1.30.x`) |
| `aws --version` | Prints AWS CLI version (e.g., `aws-cli/2.x.x Python/3.x.x Linux/...`) |
| `eksctl version` | Prints eksctl version (e.g., `0.x.x`) |

> ℹ️ The `--client` flag on `kubectl version` prevents it from attempting to contact a cluster, which is expected at this stage since the cluster has not been created yet.

---

## 5. Create the EKS Cluster

```bash
eksctl create cluster --name eks-worker --region eu-north-1 --node-type t3.medium --nodes-min 2 --nodes-max 2 --zones eu-north-1a,eu-north-1b,eu-north-1c
```

Provisions a fully managed EKS cluster with a managed node group. eksctl handles all the underlying AWS infrastructure — VPC, subnets, security groups, IAM roles, and EC2 worker nodes — automatically.

| Flag | Purpose |
|---|---|
| `--name eks-worker` | Name of the EKS cluster |
| `--region eu-north-1` | AWS region to deploy the cluster in (Stockholm) |
| `--node-type t3.medium` | EC2 instance type for the worker nodes |
| `--nodes-min 2` | Minimum number of worker nodes in the node group |
| `--nodes-max 2` | Maximum number of worker nodes — set equal to min for a fixed-size cluster |
| `--zones eu-north-1a,eu-north-1b,eu-north-1c` | Distributes worker nodes across three Availability Zones for resilience |

> ℹ️ This command typically takes **15–20 minutes** to complete. eksctl will print progress updates as it creates the CloudFormation stacks. Do not interrupt the process.

> ℹ️ Once the cluster is created, eksctl automatically updates `~/.kube/config` with the cluster credentials, so `kubectl` is immediately ready to use against the new cluster.

> ⚠️ Running this cluster will incur AWS costs. The two `t3.medium` nodes will be billed as long as the cluster is running. To delete the cluster and stop billing, run:
> ```bash
> eksctl delete cluster --name eks-worker --region eu-north-1
> ```

---

## ✅ Quick Reference

```bash
# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install AWS CLI
sudo apt-get -y install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verify all tools
kubectl version --client
aws --version
eksctl version

# Create EKS cluster
eksctl create cluster --name eks-worker --region eu-north-1 --node-type t3.medium --nodes-min 2 --nodes-max 2 --zones eu-north-1a,eu-north-1b,eu-north-1c

# Delete EKS cluster (stops AWS billing)
eksctl delete cluster --name eks-worker --region eu-north-1
```
