# Docker Installation & Setup Guide

A step-by-step guide for installing and configuring Docker on a Debian/Ubuntu-based Linux system.

---

## 1. Update Package Index

```bash
sudo apt-get -y update
```

Refreshes the local package index by fetching the latest package lists from all configured repositories. The `-y` flag automatically confirms the operation without prompting. Always run this before installing new packages to ensure you get the latest available versions.

---

## 2. Install Prerequisites & Utility Tools

```bash
sudo apt-get install -y curl wget apt-transport-https tree
```

Installs a set of essential utility packages required before setting up Docker:

| Package | Purpose |
|---|---|
| `curl` | Transfers data from URLs; used to fetch scripts and APIs |
| `wget` | Downloads files from the web via HTTP/HTTPS/FTP |
| `apt-transport-https` | Allows `apt` to fetch packages over HTTPS (required for Docker repos) |
| `tree` | Displays directory structures in a tree-like format |

---

## 3. Install Docker Engine

```bash
sudo apt-get install -y docker.io
```

Installs the `docker.io` package — the Docker Engine from Ubuntu's official package repository. This provides the core Docker daemon (`dockerd`) and the Docker CLI (`docker`), which together allow you to build, run, and manage containers on your system.

---

## 4. Enable & Start Docker Service

```bash
sudo systemctl enable --now docker
```

Configures the Docker daemon to start automatically at boot and also starts it immediately in a single command:

- `enable` — registers Docker as a systemd service that launches on system startup
- `--now` — starts the service right away without requiring a separate `systemctl start docker`

---

## 5. Add Current User to the Docker Group

```bash
sudo usermod -aG docker $USER
```

Grants your current user account permission to run Docker commands without needing `sudo` every time:

- `-a` — appends the user to the group (without removing them from other groups)
- `-G docker` — specifies the `docker` group
- `$USER` — resolves to your currently logged-in username

> ⚠️ This change takes effect after you log out and back in, or after running `newgrp docker` (see next step).

---

## 6. Apply Group Changes Without Re-login

```bash
newgrp docker
```

Activates the new `docker` group membership in the **current terminal session** immediately, without requiring a full logout/login cycle. After this command, you can run `docker` commands as a non-root user right away.

---

## 7. Install Docker Compose V2

```bash
sudo apt-get install -y docker-compose-v2
```

Installs **Docker Compose V2**, the modern rewrite of Docker Compose as a native Docker CLI plugin. It allows you to define and manage multi-container applications using a `docker-compose.yml` file.

> ℹ️ Unlike the older standalone `docker-compose` (V1), this version is invoked as `docker compose` (without the hyphen) and integrates directly into the Docker CLI.

---

## 8. Verify Docker Service Status

```bash
sudo systemctl status docker
```

Checks the current state of the Docker daemon via systemd. The output shows whether the service is **active (running)**, any recent log entries, the process ID (PID), and memory usage. Use this to confirm Docker started correctly or to diagnose issues.

---

## 9. Verify Docker Installation

```bash
docker --version
```

Prints the installed Docker Engine version to the terminal (e.g., `Docker version 27.x.x, build xxxxxxx`). A quick sanity check to confirm Docker is installed correctly and accessible from the command line.

---

## ✅ Quick Reference

```bash
# Full setup sequence
sudo apt-get -y update
sudo apt-get install -y curl wget apt-transport-https tree
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
sudo apt-get install -y docker-compose-v2

# Verify
sudo systemctl status docker
docker --version
```

---

## 🔗 Related Guides

- [Minikube Installation & Setup Guide](./Minikube.md) — local Kubernetes cluster using Docker as driver
- [kubectl Installation & Setup Guide](./Kubectl.md) — CLI tool to interact with Kubernetes clusters
- [KIND Installation & Setup Guide](./Kind.md) — lightweight local Kubernetes clusters using Docker containers
