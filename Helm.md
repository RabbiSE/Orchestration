# Helm Installation & Setup Guide

A step-by-step guide for installing and using **Helm** — the package manager for Kubernetes. Helm allows you to define, install, version, and upgrade complex Kubernetes applications using reusable packages called **charts**.

---

## ⚠️ Prerequisites

Before proceeding with Helm, ensure the following steps have been completed:

### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required before downloading Helm. `curl` is used in the next section to download the Helm installation script.

---

## 🔌 Kubernetes Cluster Requirement

> **Helm is a deployment tool — it requires a running Kubernetes cluster and kubectl to function.**

Helm communicates with your cluster through kubectl's configuration at `~/.kube/config`. Both kubectl and a Kubernetes cluster must be set up before running any `helm install` commands.

---

## 1. Download the Helm Installation Script

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
```

Downloads the official Helm installation script from the Helm GitHub repository and saves it as `get_helm.sh` in the current directory:

| Flag | Purpose |
|---|---|
| `-f` | Fails silently on server errors (no output on HTTP errors) |
| `-s` | Silent mode — suppresses progress and error output |
| `-S` | Re-enables error output even when `-s` is active, so failures are still visible |
| `-L` | Follows HTTP redirects |
| `-o get_helm.sh` | Saves the downloaded content to a file named `get_helm.sh` |

> ℹ️ The script auto-detects your OS and architecture and installs the latest stable version of Helm accordingly.

---

## 2. Make the Script Executable

```bash
chmod 700 get_helm.sh
```

Grants execute permission on the downloaded script, restricted to the **owner only**:

| Permission | Octal | Meaning |
|---|---|---|
| `700` | `rwx------` | Owner can read, write, and execute — no access for group or others |

> ℹ️ `700` is more restrictive than the `+x` used for the KIND binary, intentionally limiting execution to your user only since this is a shell script that runs with elevated privileges.

---

## 3. Run the Installation Script

```bash
./get_helm.sh
```

Executes the Helm installation script from the current directory. The script will automatically download and install the latest stable Helm binary to `/usr/local/bin/helm`, making it available system-wide as a CLI command.

> ℹ️ The script may prompt for your `sudo` password as it installs Helm to a system directory. After successful installation, the `get_helm.sh` script can be safely removed with `rm get_helm.sh`.

---

## 4. Create a New Chart

```bash
helm create chartname
```

Scaffolds a new Helm chart with a standard directory structure in a folder named `chartname`. Replace `chartname` with your desired chart name. The generated structure looks like this:

```
chartname/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values for the chart
├── charts/             # Directory for chart dependencies
└── templates/          # Kubernetes manifest templates (Deployments, Services, etc.)
```

Edit the files inside `templates/` and `values.yaml` to define your application's Kubernetes resources and configuration.

---

## 5. Package the Chart

```bash
helm package chartname/
```

Compresses the chart directory into a versioned `.tgz` archive (e.g., `chartname-0.1.0.tgz`) in the current directory. The version is taken from the `version` field in `Chart.yaml`.

Packaging is used to:
- Share charts with others or publish them to a Helm repository
- Version and archive a specific release of your chart before deployment
- Ensure the chart is validated and properly structured before distribution

---

## 6. Install a Chart (First Time)

```bash
helm install installationname chartname --namespace=namespace --create-namespace
```

Deploys the chart onto the Kubernetes cluster for the first time as a named **release**:

| Flag/Argument | Purpose |
|---|---|
| `installationname` | A unique name for this release — used to identify, upgrade, and uninstall it later |
| `chartname` | The chart to install (local directory or `.tgz` package) |
| `--namespace=namespace` | Deploys all chart resources into the specified Kubernetes namespace |
| `--create-namespace` | Automatically creates the namespace if it does not already exist |

> ℹ️ Replace `installationname`, `chartname`, and `namespace` with your actual release name, chart name, and target namespace.

> ⚠️ Use `--create-namespace` only on the **first install**. Subsequent upgrades do not require it as the namespace already exists (see Step 7).

---

## 7. Upgrade a Release

```bash
helm install installationname chartname --namespace=namespace
```

> ⚠️ Before running this command, ensure you have made the necessary changes to your chart (e.g., updated `values.yaml`, modified templates, or bumped the version in `Chart.yaml`), then repackage the chart using `helm package chartname/` (Step 5).

Re-deploys an updated version of the chart to an existing namespace. This is used when a new packaging or release upgrade is needed:

- **`--create-namespace` is omitted** — the namespace already exists from the initial install
- Helm records each install and upgrade as a new **revision**, enabling rollback if needed
- All previously deployed resources are updated in-place with the new chart version

> ℹ️ For a cleaner upgrade workflow, consider using `helm upgrade installationname chartname --namespace=namespace` which is purpose-built for updating existing releases and preserves release history more explicitly.

---

## 8. Roll Back a Release

```bash
helm rollback installationname revnumber --namespace=namespace
```

Reverts the specified release to a previous revision. Helm stores a full history of every install and upgrade, making rollbacks straightforward:

| Argument | Purpose |
|---|---|
| `installationname` | The name of the release to roll back |
| `revnumber` | The revision number to roll back to (e.g., `1`, `2`, `3`) |
| `--namespace=namespace` | The namespace where the release is deployed |

To view the revision history of a release before rolling back:

```bash
helm history installationname --namespace=namespace
```

This lists all revisions with their status, timestamps, and chart versions — helping you identify which revision to target.

> ℹ️ Rolling back itself creates a new revision in the history rather than deleting the failed one, keeping the audit trail intact.

---

## 9. Uninstall a Release

```bash
helm uninstall installationname
```

Removes the specified Helm release and deletes all associated Kubernetes resources (Deployments, Services, ConfigMaps, etc.) that were created by the chart. This is a **clean uninstall** — unlike `kubectl delete`, Helm knows exactly which resources belong to the release.

> ⚠️ This is a **destructive operation** — all resources tied to the release are permanently removed. Add `--namespace=namespace` if the release was deployed in a specific namespace.

> ℹ️ By default, release history is also deleted. To retain history for auditing purposes, use `helm uninstall installationname --keep-history`.

---

## ✅ Quick Reference

```bash
# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree

# Download and run Helm installer
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh

# Create and package a chart
helm create chartname
helm package chartname/

# First install (creates namespace)
helm install installationname chartname --namespace=namespace --create-namespace

# Upgrade (after repackaging the chart)
helm package chartname/
helm install installationname chartname --namespace=namespace

# Roll back to a previous revision
helm history installationname --namespace=namespace
helm rollback installationname revnumber --namespace=namespace

# Uninstall a release
helm uninstall installationname
```
