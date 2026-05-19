# Chapter 2: Server Setup & Docker Installation

In this chapter, you will prepare your Ubuntu 22.04 server and install Docker Engine with Docker Compose.

---

## Step 1: SSH Into Your Server

Open a terminal on your local machine and connect to your server:

```bash
ssh username@your-server-ip
```

Replace `username` with your actual username and `your-server-ip` with the server's IP address.

> If this is a fresh cloud server (AWS EC2, DigitalOcean, Linode, etc.), you will likely SSH in as `root` and create a user afterward. For a local VM, use whatever user you configured during setup.

---

## Step 2: Update System Packages

Always start with a full system update:

```bash
sudo apt update && sudo apt upgrade -y
```

**What this does:** `apt update` refreshes the package index from all configured repositories. `apt upgrade` installs the latest versions of all installed packages.

> If a kernel update was installed, consider rebooting: `sudo reboot`

---

## Step 3: Install Prerequisite Packages

```bash
sudo apt install -y \
  curl \
  git \
  unzip \
  ca-certificates \
  gnupg \
  lsb-release
```

**What these are:**

| Package | Purpose |
|---|---|
| `curl` | Download files / API calls |
| `git` | Clone source code |
| `unzip` | Extract archives |
| `ca-certificates` | SSL/TLS certificate verification |
| `gnupg` | GPG key verification |
| `lsb-release` | Linux Standard Base info (used by Docker setup) |

---

## Step 4: Add Docker's Official GPG Key

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

**What this does:** Downloads Docker's GPG key, converts it to the correct format, and stores it so `apt` can verify Docker packages are authentic.

---

## Step 5: Add Docker's APT Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**What this does:** Adds the official Docker repository to `apt`'s sources so you can install Docker Engine directly.

---

## Step 6: Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**What each package is:**

| Package | Purpose |
|---|---|
| `docker-ce` | Docker Engine (the core container runtime) |
| `docker-ce-cli` | Docker command-line interface |
| `containerd.io` | Container runtime (manages container lifecycle) |
| `docker-buildx-plugin` | Advanced build features (multi-architecture, etc.) |
| `docker-compose-plugin` | Docker Compose v2 (runs as a `docker compose` subcommand) |

**Note:** Docker Compose v2 is now a Docker plugin. You run it with `docker compose` (no hyphen) rather than the old standalone `docker-compose`.

---

## Step 7: Enable and Start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

**What this does:**
- `enable` — Docker will start automatically on system boot
- `start` — Starts Docker right now

---

## Step 8: Add Your User to the Docker Group

```bash
sudo usermod -aG docker $USER
```

**What this does:** Adds your user to the `docker` group so you can run Docker commands without needing `sudo` every time.

> **Important:** After this command, log out and back in (or reconnect SSH) for the group change to take effect:
>
> ```bash
> exit
> ssh username@your-server-ip
> ```
>
> You can also run `newgrp docker` to activate the change in the current session without reconnecting.

---

## Step 9: Verify the Installation

```bash
docker --version
docker compose version
```

Expected output (versions will differ):

```
Docker version 26.x.x, build xxxxxxx
Docker Compose version v2.x.x-desktop.x
```

---

## Step 10: Test That Docker Works

Run the classic "hello world" container:

```bash
docker run hello-world
```

Expected output includes:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## Verification Checklist

Before moving on, confirm:

- [x] `docker --version` prints a version number
- [x] `docker compose version` prints a version number
- [x] `docker run hello-world` runs without errors
- [x] `docker ps` shows no errors (the `hello-world` container exits immediately, so it will not appear in the list — that is expected)

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `permission denied` when running Docker commands | Log out and back in (the `docker` group change needs a new session). Run `newgrp docker` as a temporary workaround. |
| `docker: command not found` | Docker installation may have failed. Run `sudo apt install -y docker-ce docker-ce-cli containerd.io` again. |
| `Cannot connect to the Docker daemon` | Docker is not running. Run `sudo systemctl start docker` and then `sudo systemctl status docker` to check. |

---

**Next:** [Chapter 3 — Project Structure & Dockerfile](03-project-structure-and-dockerfile.md)
