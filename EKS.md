# Amazon EKS Installation & Setup Guide

A step-by-step guide for provisioning and connecting to a managed Kubernetes cluster on **Amazon EKS** (Elastic Kubernetes Service) from an EC2 instance. EKS is AWS's fully managed Kubernetes service — AWS handles the control plane, and you manage the worker nodes.

---

## ⚠️ Prerequisites

Before proceeding, ensure the following steps have been completed:

### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required throughout this guide:

| Package | Purpose |
|---|---|
| `curl` | Downloads binaries, scripts, and keys from URLs |
| `wget` | Alternative file downloader via HTTP/HTTPS/FTP |
| `apt-transport-https` | Allows `apt` to fetch packages over HTTPS |
| `tree` | Displays directory structures in a tree-like format |

---

## 1. Install kubectl

### Download the kubectl Binary

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Downloads the latest stable kubectl binary for 64-bit Linux. This command uses two nested `curl` calls:

- **Inner command** `$(curl -L -s https://dl.k8s.io/release/stable.txt)` — fetches the latest stable Kubernetes version string (e.g., `v1.30.0`) from the official Kubernetes release API
- **Outer command** `curl -LO "https://dl.k8s.io/release/.../kubectl"` — uses that version string to construct the full download URL and fetch the binary

| Flag | Purpose |
|---|---|
| `-L` | Follows HTTP redirects |
| `-O` | Saves the file with its original filename (`kubectl`) |
| `-s` | Silent mode — suppresses progress output for the inner curl |

### Install kubectl System-Wide

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Installs the downloaded binary to `/usr/local/bin/` with secure, correct permissions:

| Flag | Purpose |
|---|---|
| `-o root` | Sets the file **owner** to `root` |
| `-g root` | Sets the file **group** to `root` |
| `-m 0755` | Sets file permissions to `rwxr-xr-x` — owner can read/write/execute, everyone else can read/execute |

> ℹ️ After installation, the downloaded `kubectl` file in your current directory can be safely removed with `rm kubectl`.

---

## 2. Create and Attach IAM Role to EC2

> ⚠️ **This step must be completed in the AWS Console before proceeding. eksctl and the AWS CLI require AWS permissions to create and manage EKS resources.**

### Create the IAM Role

1. Go to the **AWS Console** → **IAM** → **Roles** → **Create Role**
2. Select **Trusted entity type**: `AWS Service`
3. Select **Use case**: `EC2`
4. Click **Next** and attach the **`AdministratorAccess`** policy
5. Give the role a name (e.g., `eks-ec2-admin-role`) and click **Create Role**

### Attach the Role to Your EC2 Instance

1. Go to **EC2** → **Instances** → select your instance
2. Click **Actions** → **Security** → **Modify IAM Role**
3. Select the role you just created and click **Update IAM Role**

> ℹ️ The IAM role grants the EC2 instance permission to call AWS APIs (EKS, EC2, CloudFormation, IAM, etc.) without needing to configure static access keys. eksctl relies on this role to provision all cluster resources on your behalf.

> ⚠️ `AdministratorAccess` grants full AWS permissions and is suitable for development and learning environments. For production, create a scoped IAM policy with only the permissions eksctl requires.

---

## 3. Install the AWS CLI

```bash
sudo apt-get install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Installs the **AWS CLI v2** — the command-line tool for interacting with AWS services. It is required by eksctl to authenticate and make API calls to AWS.

| Command | Purpose |
|---|---|
| `sudo apt-get install -y unzip` | Installs `unzip`, required to extract the downloaded archive |
| `curl ... -o "awscliv2.zip"` | Downloads the AWS CLI v2 installer as a zip archive |
| `unzip awscliv2.zip` | Extracts the installer to an `aws/` directory in the current folder |
| `sudo ./aws/install` | Runs the installer, placing the `aws` binary in `/usr/local/bin/` |

> ℹ️ After installation, you can clean up with `rm -rf awscliv2.zip aws/`. Since the EC2 instance has an IAM role attached (Step 2), no manual `aws configure` is needed — the CLI automatically uses the instance's role credentials.

---

## 4. Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

Installs **eksctl** — the official CLI tool for creating and managing EKS clusters. Two commands chained together:

**Download and extract:**

| Flag/Part | Purpose |
|---|---|
| `--silent` | Suppresses progress output |
| `--location` | Follows HTTP redirects |
| `$(uname -s)` | Dynamically inserts the OS name (e.g., `Linux`) to select the correct binary |
| `tar xz -C /tmp` | Extracts the `.tar.gz` archive directly into `/tmp` without saving the archive file |

**Install:**
- `sudo mv /tmp/eksctl /usr/local/bin` — moves the extracted binary to `/usr/local/bin/`, making it available system-wide as a CLI command

---

## 5. Verify Installations

```bash
kubectl version --client
aws --version
eksctl version
```

Confirms all three tools are installed correctly and accessible from the command line:

| Command | Expected Output |
|---|---|
| `kubectl version --client` | `Client Version: v1.xx.x` |
| `aws --version` | `aws-cli/2.x.x Python/3.x.x ...` |
| `eksctl version` | `0.x.x` |

> ℹ️ The `--client` flag on kubectl ensures it only checks the local binary version without attempting to contact a cluster, which is not yet set up at this stage.

---

## 6. Create the EKS Cluster

```bash
eksctl create cluster \
  --name eks-worker \
  --region eu-north-1 \
  --node-type t3.medium \
  --nodes-min 2 \
  --nodes-max 2 \
  --zones eu-north-1a,eu-north-1b,eu-north-1c
```

Provisions a fully managed EKS cluster on AWS. eksctl orchestrates the creation of all required AWS resources including the EKS control plane, VPC, subnets, security groups, and EC2 worker node instances via CloudFormation.

| Flag | Purpose |
|---|---|
| `--name eks-worker` | The name of the EKS cluster |
| `--region eu-north-1` | The AWS region to deploy the cluster in (Stockholm) |
| `--node-type t3.medium` | The EC2 instance type for worker nodes — `t3.medium` provides 2 vCPU and 4GB RAM |
| `--nodes-min 2` | Minimum number of worker nodes in the Auto Scaling Group |
| `--nodes-max 2` | Maximum number of worker nodes in the Auto Scaling Group |
| `--zones eu-north-1a,eu-north-1b,eu-north-1c` | Distributes worker nodes across three Availability Zones for high availability |

> ℹ️ Cluster creation typically takes **15–25 minutes**. eksctl will stream progress to the terminal as it creates each resource. Do not interrupt the process.

> ℹ️ Once the cluster is created, eksctl automatically updates your `~/.kube/config` file with the cluster's connection details, so kubectl is immediately ready to use against the new cluster.

> ⚠️ Running worker nodes incurs AWS costs. Remember to delete the cluster when no longer needed (see Quick Reference below).

---

## ✅ Quick Reference

```bash
# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# [AWS Console] Create IAM role with AdministratorAccess and attach to EC2 instance

# Install AWS CLI
sudo apt-get install -y unzip
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
eksctl create cluster \
  --name eks-worker \
  --region eu-north-1 \
  --node-type t3.medium \
  --nodes-min 2 \
  --nodes-max 2 \
  --zones eu-north-1a,eu-north-1b,eu-north-1c

# Delete EKS cluster (when no longer needed)
eksctl delete cluster --name eks-worker --region eu-north-1
```
