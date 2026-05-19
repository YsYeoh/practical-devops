# Chapter 6: Jenkins Installation & Setup

In this chapter, you will install Jenkins on your Ubuntu server, complete the initial setup, and install the necessary plugins.

---

## Step 1: Install OpenJDK 17

Jenkins requires Java to run. Install OpenJDK 17:

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

Verify the installation:

```bash
java --version
```

Expected output:

```
openjdk 17.x.x ...
```

> Jenkins has specific Java version requirements. As of 2024-2025, Jenkins LTS supports Java 17 and Java 21. Java 17 is the safest choice for this guide.

---

## Step 2: Add the Jenkins Repository

Jenkins is not in Ubuntu's default repositories, so you need to add the official Jenkins repository.

First, import the Jenkins GPG key:

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

Then add the repository:

```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

---

## Step 3: Install Jenkins

```bash
sudo apt update
sudo apt install jenkins -y
```

---

## Step 4: Start and Enable Jenkins

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Check that Jenkins is running:

```bash
sudo systemctl status jenkins
```

Expected output: `active (running)`

---

## Step 5: Configure the Firewall (If UFW Is Enabled)

If you have UFW active (or plan to enable it later), allow Jenkins on port 8080:

```bash
sudo ufw allow 8080
```

> You will also need to allow port 8080 in your cloud provider's firewall / security group if applicable (AWS Security Group, DigitalOcean Firewall, etc.).

---

## Step 6: Access Jenkins for the First Time

Open your browser and navigate to:

```
http://YOUR_SERVER_IP:8080
```

You should see the "Unlock Jenkins" page asking for an initial admin password.

---

## Step 7: Retrieve the Initial Admin Password

On your server, run:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the password (a UUID-like string) and paste it into the Jenkins web UI.

---

## Step 8: Install Suggested Plugins

On the "Customize Jenkins" page:

1. Click **Install suggested plugins**
2. Wait for the installation to complete (this may take a few minutes)

The suggested plugins include:

| Plugin | Purpose |
|---|---|
| Git Plugin | Clone repositories from GitHub |
| Pipeline Plugin | Define CI/CD pipelines as code |
| Credentials Plugin | Store GitHub tokens and SSH keys securely |
| Docker Pipeline Plugin | Build and push Docker images from pipelines |
| GitHub Integration | Webhook connectivity |

> If the installation fails for some plugins, do not worry — you can install them manually later. The most critical plugin for this guide is **Git Plugin**.

---

## Step 9: Create the Admin User

After plugins are installed:

1. Fill in the admin user details (username, password, full name, email)
2. Click **Save and Finish**
3. Click **Start using Jenkins**

---

## Step 10: Verify Jenkins Is Ready

On the Jenkins dashboard:

- You should see the Jenkins logo and "Welcome to Jenkins!"
- The URL in the address bar should show `http://YOUR_SERVER_IP:8080`
- Navigate to **Manage Jenkins > System** to verify the configuration

---

## Step 11: Install Any Missing Plugins (Optional)

To install additional plugins later:

```bash
Manage Jenkins → Plugins → Available plugins
```

Plugins you may want for this guide:

- **Blue Ocean** — A modern, visual pipeline editor
- **Docker Pipeline** — Build and use Docker containers from pipelines
- **GitHub Integration** — Better GitHub webhook support

---

## Verification Checklist

Before moving on, confirm:

- [x] `java --version` shows Java 17
- [x] Jenkins is running: `sudo systemctl status jenkins` shows `active (running)`
- [x] Jenkins web UI is accessible at `http://YOUR_SERVER_IP:8080`
- [x] You have logged in with the initial admin password and created an admin user
- [x] Suggested plugins are installed

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `java: command not found` | Java installation failed. Run `sudo apt install openjdk-17-jdk -y` again |
| Jenkins web UI not loading | Check if Jenkins is running: `sudo systemctl status jenkins`. Check port 8080 is open: `sudo ufw status` |
| `Failed to connect to repository` during plugin install | Jenkins needs internet access. Check your server's network connectivity |
| Jenkins service fails to start | Check logs: `sudo journalctl -u jenkins -n 50` |
| Port 8080 is already in use | Find the process: `sudo lsof -i :8080`. Stop it or configure Jenkins to use a different port in `/etc/default/jenkins` |

---

**Next:** [Chapter 7 — GitHub Integration & CI/CD Pipeline](07-github-integration-and-pipeline.md)
