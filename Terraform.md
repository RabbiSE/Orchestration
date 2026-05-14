# Terraform Installation & Setup Guide

A step-by-step guide for installing **Terraform** — an open-source Infrastructure as Code (IaC) tool by HashiCorp that allows you to define, provision, and manage cloud infrastructure using declarative configuration files.

---

## ⚠️ Prerequisites

Before proceeding with Terraform, ensure the following steps have been completed:

### 1. Update Package Index & Install Prerequisites

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
```

Updates the package index and installs two required utilities in a single command:

| Package | Purpose |
|---|---|
| `gnupg` | GNU Privacy Guard — required to import and verify the HashiCorp GPG signing key |
| `software-properties-common` | Provides tools for managing additional apt repositories (e.g., `add-apt-repository`) |

---

## 1. Add the HashiCorp GPG Key

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
```

Downloads and stores the official HashiCorp GPG signing key. This is a three-step pipeline:

| Step | Command | Purpose |
|---|---|---|
| 1 | `wget -O-` | Downloads the GPG key from HashiCorp's official apt releases page and outputs it to stdout instead of saving to a file |
| 2 | `gpg --dearmor` | Converts the key from ASCII-armored format (`.asc`) to binary format (`.gpg`) that apt can use |
| 3 | `sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg` | Saves the converted binary key to the system keyrings directory |

> ℹ️ The `> /dev/null` at the end suppresses `tee`'s terminal output — the file is still written correctly.

---

## 2. Verify the GPG Key Fingerprint

```bash
gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint
```

Reads and displays the fingerprint of the newly added HashiCorp GPG key to verify it is authentic before proceeding with installation.

| Flag | Purpose |
|---|---|
| `--no-default-keyring` | Prevents GPG from loading the default user keyring — only the specified keyring is read |
| `--keyring` | Points to the HashiCorp keyring file saved in Step 1 |
| `--fingerprint` | Prints the full fingerprint of all keys in the specified keyring |

> ℹ️ The expected fingerprint for HashiCorp's key is `798A EC65 4E5C 1542 8C8E 42EE AA16 FCBC A621 E701`. Verify the output matches before continuing.

---

## 3. Add the HashiCorp Repository

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
```

Adds the official HashiCorp apt repository to the system's package sources so that `apt` can install and update Terraform directly.

| Part | Purpose |
|---|---|
| `signed-by=...` | Tells apt to verify packages from this repo using the HashiCorp GPG key saved in Step 1 |
| `$(lsb_release -cs)` | Dynamically inserts the Ubuntu/Debian codename (e.g., `jammy`, `focal`) to match the correct repo branch |
| `sudo tee /etc/apt/sources.list.d/hashicorp.list` | Writes the repository entry to a dedicated file in the apt sources directory |

---

## 4. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the package index again to pick up the newly added HashiCorp repository from Step 3. This is required before `apt` can find and install the `terraform` package.

---

## 5. Install Terraform

```bash
sudo apt-get install -y terraform
```

Installs the latest stable version of Terraform from the official HashiCorp repository. The binary is placed in `/usr/bin/terraform`, making it available system-wide as a CLI command.

---

## 6. Verify Terraform Installation

```bash
terraform --version
```

Prints the installed Terraform version to the terminal (e.g., `Terraform v1.x.x`). Confirms the binary was installed correctly and is accessible from the command line.

---

## ✅ Quick Reference

```bash
# Install prerequisites
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

# Verify GPG key fingerprint
gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update package index
sudo apt-get -y update

# Install Terraform
sudo apt-get install -y terraform

# Verify
terraform --version
```
