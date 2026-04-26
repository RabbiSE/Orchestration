# Jenkins Installation & Setup Guide

A step-by-step guide for installing and configuring **Jenkins** — an open-source automation server used for building, testing, and deploying applications. This guide covers setting up both a **Jenkins Master** (controller) node and a **Jenkins Agent** (worker) node, and connecting them via SSH.

---

## 🖥️ Jenkins Master Node

The Master node is the central controller — it schedules builds, manages configurations, and serves the Jenkins web UI on port `8080`.

---

### ⚠️ Prerequisites

#### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available. This must be run before installing any packages to avoid resolving outdated or missing dependencies.

#### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities required before setting up Jenkins. `wget` is used in the next section to download the Jenkins GPG signing key.

---

### 1. Install Java

```bash
sudo apt-get -y install fontconfig openjdk-21-jre
```

Jenkins is a Java application and requires a Java Runtime Environment (JRE) to run. Two packages are installed here:

| Package | Purpose |
|---|---|
| `fontconfig` | Required by Jenkins for font rendering in the web UI and report generation |
| `openjdk-21-jre` | OpenJDK 21 Java Runtime Environment — the version required for Jenkins LTS releases |

> ⚠️ Jenkins will fail to start if Java is not installed. OpenJDK 21 is the currently recommended version for Jenkins LTS.

---

### 2. Add the Jenkins Repository

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Adds the official Jenkins Debian repository so `apt` can install and update Jenkins directly. Two actions are performed here:

**Download and store the GPG signing key:**
`sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc` downloads the Jenkins repository GPG key and saves it to `/etc/apt/keyrings/`. This key is used by `apt` to verify the authenticity of packages from the Jenkins repository.

**Add the Jenkins apt repository:**
The `echo ... | sudo tee` command writes the Jenkins stable repository entry to `/etc/apt/sources.list.d/jenkins.list`. The `signed-by` reference points to the key saved in the previous step, ensuring all installed packages are cryptographically verified.

> ℹ️ The `> /dev/null` at the end suppresses the `tee` output to keep the terminal clean — the file is still written correctly.

---

### 3. Install Jenkins

```bash
sudo apt-get -y update
sudo apt-get -y install jenkins
```

- **`sudo apt-get -y update`** — refreshes the package index again to pick up the newly added Jenkins repository from Step 2
- **`sudo apt-get -y install jenkins`** — installs the latest Jenkins LTS release from the official Jenkins repository

---

### 4. Enable & Check Jenkins Service

```bash
systemctl status jenkins
sudo systemctl enable jenkins
```

| Command | Purpose |
|---|---|
| `systemctl status jenkins` | Checks whether the Jenkins service is currently running and shows recent logs |
| `sudo systemctl enable jenkins` | Registers Jenkins as a systemd service to start automatically on every system reboot |

> ℹ️ Jenkins starts automatically after installation. If the status shows it is not running, start it manually with `sudo systemctl start jenkins`.

---

### 5. Open Port 8080

> **Expose port `8080` in your EC2 Security Group to access the Jenkins web UI.**

Jenkins serves its web interface on port `8080` by default. This port must be open to inbound traffic in your EC2 instance's Security Group.

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| `8080` | TCP | Inbound | Jenkins web UI |

To open the port:
1. Navigate to **EC2 → Security Groups** in the AWS Console
2. Select the Security Group attached to your Jenkins instance
3. Click **Edit inbound rules → Add rule**
4. Set Type: **Custom TCP**, Port: **8080**, Source: your IP or `0.0.0.0/0`

> ⚠️ Using `0.0.0.0/0` opens the port to the entire internet. For security, restrict access to your own IP address.

---

### 6. Retrieve the Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

On the **first-time installation only**, Jenkins generates a one-time admin password stored at `/var/lib/jenkins/secrets/initialAdminPassword`. This password is required to unlock Jenkins when you first open the web UI in a browser.

**Steps to complete the initial setup:**
1. Open a browser and navigate to `http://<your-ec2-public-ip>:8080`
2. Paste the password retrieved from the command above into the **Administrator password** field
3. Follow the setup wizard to install suggested plugins and create your admin user

> ℹ️ This file is only needed once. After the initial setup is complete, Jenkins uses the credentials you configured during the wizard.

---

## 👷 Jenkins Agent Node

The Agent node is a worker machine that executes build jobs delegated by the Master. Agents allow Jenkins to distribute workloads across multiple machines.

---

### ⚠️ Prerequisites

#### 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index to ensure the latest package versions are available.

#### 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs essential utilities on the agent node.

---

### 1. Install Java

```bash
sudo apt-get -y install fontconfig openjdk-21-jre
```

The Jenkins agent process runs as a Java application and requires the same JRE version as the Master. Install OpenJDK 21 on every agent node.

| Package | Purpose |
|---|---|
| `fontconfig` | Required for font rendering used by Jenkins build reports and UI components |
| `openjdk-21-jre` | Java Runtime Environment — required to run the Jenkins agent process |

---

### 2. Verify Java Installation

```bash
java -version
```

Confirms Java is installed correctly on the agent node. The output should display the OpenJDK 21 version, for example:

```
openjdk version "21.x.x" ...
OpenJDK Runtime Environment ...
```

> ⚠️ If `java -version` returns an error or shows an incompatible version, the agent will fail to connect to the Master. Ensure `openjdk-21-jre` is installed before proceeding.

---

## 🔗 Connecting Master and Agent via SSH

Jenkins communicates with Agent nodes over SSH. The Master authenticates to the Agent using an SSH key pair — the private key is stored in Jenkins credentials, and the public key is placed on the Agent.

---

### Step 1 — Generate SSH Keys on the Master Node

Run the following on the **Jenkins Master**:

```bash
ssh-keygen -t rsa
```

Generates an RSA SSH key pair in `~/.ssh/`:

| File | Purpose |
|---|---|
| `~/.ssh/id_rsa` | **Private key** — stays on the Master, will be added to Jenkins credentials |
| `~/.ssh/id_rsa.pub` | **Public key** — must be copied to the Agent's `authorized_keys` file |

> ℹ️ Press `Enter` at all prompts to accept the default file location and no passphrase, unless you want to configure a passphrase for additional security.

---

### Step 2 — Add the Private Key to Jenkins Credentials

On the **Jenkins Master** web UI, add the SSH private key so Jenkins can authenticate to the Agent:

1. Navigate to **Manage Jenkins → Credentials**
2. Select the appropriate scope (e.g., **Global**)
3. Click **Add Credentials**
4. Select **SSH Username with private key**
5. Set **Username** to the OS user on the Agent (e.g., `ubuntu`)
6. Under **Private Key**, select **Enter directly** and paste the contents of `~/.ssh/id_rsa`:

```bash
cat ~/.ssh/id_rsa
```

7. Give the credential a meaningful **ID** (e.g., `jenkins-agent-ssh`) and **Description**
8. Click **Create**

> ℹ️ The Username must match the OS-level user account on the Agent machine that owns the `authorized_keys` file configured in Step 3.

---

### Step 3 — Add the Public Key to the Agent

On the **Jenkins Agent**, append the Master's public key to the `authorized_keys` file to allow the Master to SSH in.

First, view the public key on the **Master**:

```bash
cat ~/.ssh/id_rsa.pub
```

Then on the **Agent**, open the `authorized_keys` file:

```bash
vi ~/.ssh/authorized_keys
```

Paste the public key on a new line, then save and exit with `:wq`.

> ⚠️ Each public key must be on its own single line in `authorized_keys`. Ensure there are no line breaks within the key itself.

> ℹ️ If the `~/.ssh/` directory or `authorized_keys` file does not exist on the Agent, create them first:
> ```bash
> mkdir -p ~/.ssh && chmod 700 ~/.ssh
> touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
> ```

---

### Step 4 — Create the Agent Node in Jenkins

Back on the **Jenkins Master** web UI, register the Agent:

1. Navigate to **Manage Jenkins → Nodes → New Node**
2. Give the node a name (e.g., `agent-1`) and select **Permanent Agent**, then click **Create**
3. Fill in the following fields:

| Field | Value | Purpose |
|---|---|---|
| **Remote root directory** | e.g., `/home/ubuntu/jenkins` | Directory on the Agent where Jenkins stores workspace files and build data |
| **Labels** | e.g., `linux`, `build`, `docker` | Tags used in pipeline jobs to target this specific agent (e.g., `agent { label 'linux' }`) |
| **Launch method** | **Launch agents via SSH** | Tells Jenkins to connect to the Agent using SSH |
| **Host** | Agent's private IP address | The IP address Jenkins will SSH into to start the agent process |
| **Credentials** | Select the credential created in Step 2 | The SSH private key Jenkins uses to authenticate with the Agent |
| **Host Key Verification Strategy** | **Non-verifying** (lab) / **Known hosts** (production) | Controls whether Jenkins verifies the Agent's SSH host key |

4. Click **Save** — Jenkins will immediately attempt to connect to the Agent via SSH and launch the agent process

> ℹ️ After saving, check the node's log under **Manage Jenkins → Nodes → agent-1 → Log** to confirm the connection was successful. A healthy agent will show `Agent successfully connected and online`.

> ⚠️ If the connection fails, verify that port `22` is open in the Agent's Security Group and that the public key was correctly added to `authorized_keys` in Step 3.

---

## ✅ Quick Reference

```bash
# ---- JENKINS MASTER ----

# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree

# Install Java
sudo apt-get -y install fontconfig openjdk-21-jre

# Add Jenkins repository
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt-get -y update
sudo apt-get -y install jenkins

# Enable and check service
systemctl status jenkins
sudo systemctl enable jenkins

# Retrieve initial admin password (first-time setup only)
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# ---- JENKINS AGENT ----

# Prerequisites
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree

# Install Java
sudo apt-get -y install fontconfig openjdk-21-jre

# Verify Java
java -version

# ---- SSH KEY SETUP (run on Master) ----

# Generate SSH key pair
ssh-keygen -t rsa

# View private key (paste into Jenkins Credentials)
cat ~/.ssh/id_rsa

# View public key (paste into Agent's authorized_keys)
cat ~/.ssh/id_rsa.pub
```
