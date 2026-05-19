# Chapter 7: GitHub Integration & CI/CD Pipeline

In this chapter, you will connect Jenkins to your GitHub repository, create a `Jenkinsfile` pipeline, and set up automatic deployment triggered by Git pushes.

---

## Part 1: Create a Jenkins Pipeline Job

### Step 1.1 — Create a New Pipeline Item

1. Log into Jenkins web UI
2. Click **New Item** (top-left)
3. Enter a name: `practical-devops-pipeline`
4. Select **Pipeline**
5. Click **OK**

### Step 1.2 — Configure the Pipeline

On the pipeline configuration page:

1. Scroll to the **Pipeline** section
2. From the **Definition** dropdown, select **Pipeline script from SCM**
3. From the **SCM** dropdown, select **Git**
4. In **Repository URL**, paste your GitHub repository URL:

   ```
   https://github.com/YOUR_USERNAME/YOUR_REPO.git
   ```

5. In **Branches to build**, enter `*/main` (or your default branch name)
6. **Script Path** should remain as `Jenkinsfile`

Do not click Save yet — you need to add credentials first.

---

## Part 2: Add GitHub Credentials to Jenkins

### Step 2.1 — Generate a GitHub Personal Access Token

1. Go to GitHub → **Settings** (your profile menu, top-right) → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Give it a name: `Jenkins CI`
4. Set expiration (choose a reasonable period, e.g., 90 days)
5. Select these scopes:

   | Scope | Why |
   |---|---|
   | `repo` (full control) | Access private repositories and read public ones |
   | `admin:repo_hook` | Create webhooks automatically |

6. Click **Generate token**
7. **Copy the token immediately** — GitHub will not show it again

### Step 2.2 — Add Token to Jenkins Credentials

1. In Jenkins, go to **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)** → **Add Credentials**
2. Fill in:

   | Field | Value |
   |---|---|
   | Kind | **Username with password** |
   | Username | Your GitHub username |
   | Password | The personal access token (not your GitHub password) |
   | ID | `github-token` |
   | Description | `GitHub Personal Access Token` |

3. Click **Create**

### Step 2.3 — Select the Credential in the Pipeline Config

1. Go back to your pipeline configuration (`practical-devops-pipeline`)
2. In the **Pipeline** section, next to **Repository URL**, click **Add** → select `github-token`
3. Click **Save**

---

## Part 3: Create the Jenkinsfile

Now create the pipeline definition file in your project root on your server (or on your local machine, commit it, and push to GitHub):

```bash
nano Jenkinsfile
```

Paste the following:

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'YOUR_GITHUB_REPO'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Deploy Containers') {
            steps {
                sh 'docker compose up -d'
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker image prune -f'
            }
        }
    }
}
```

**Important:** Replace `YOUR_GITHUB_REPO` with your actual repository URL.

### Understanding the Pipeline Stages

| Stage | Purpose |
|---|---|
| **Clone Repository** | Pulls the latest code from the `main` branch of your GitHub repo |
| **Build Docker Images** | Runs `docker compose build` to rebuild images if the Dockerfile or app code changed |
| **Deploy Containers** | Runs `docker compose up -d` to start (or update) running containers |
| **Cleanup** | Removes unused Docker images with `docker image prune -f` to free disk space |

### Why the Cleanup Stage Matters

Docker storage grows quickly. Every time Jenkins rebuilds your application:

- Old Docker images remain on disk
- Unused layers accumulate
- Over time, disk space fills up

The `prune -f` command removes:
- Dangling images (layers no longer associated with any tagged image)
- Unused containers, networks, and build cache (with `-a` flag)

> **Pro tip:** Add `docker system prune -af` for a more aggressive cleanup, but be aware this removes all stopped containers and unused networks too.

---

## Part 4: Test the Pipeline

### Step 4.1 — Initial Manual Run

1. In Jenkins, go to your pipeline (`practical-devops-pipeline`)
2. Click **Build Now** (left sidebar)
3. Click on the build number (e.g., `#1`) in the **Build History**
4. Click **Console Output** to watch the pipeline run

Expected output:

```
Cloning repository https://github.com/YOUR_USERNAME/YOUR_REPO.git
...
Building Docker images...
...
Deploying containers...
...
Pruning unused images...
...
Finished: SUCCESS
```

### Step 4.2 — Fixing Permission Issues

If you see `permission denied` when Jenkins tries to run Docker commands:

```bash
# Add the Jenkins user to the docker group
sudo usermod -aG docker jenkins

# Restart Jenkins to apply the change
sudo systemctl restart jenkins
```

> This is required because Jenkins runs as the `jenkins` user, which does not have permission to access the Docker daemon by default.

---

## Part 5: Set Up Automatic Deployment with Webhooks

Now configure GitHub to notify Jenkins whenever you push code.

### Step 5.1 — Add a Webhook in GitHub

1. Go to your GitHub repository → **Settings** → **Webhooks** → **Add webhook**
2. Fill in:

   | Field | Value |
   |---|---|
   | Payload URL | `http://YOUR_JENKINS_SERVER:8080/github-webhook/` |
   | Content type | `application/json` |
   | Events | **Just the push event** (or select "Send me everything") |

3. Click **Add webhook**

> **Note:** The trailing slash in `github-webhook/` is required.

### Step 5.2 — Verify the Webhook

After adding the webhook, GitHub sends a test ping. In the webhook settings page, you should see a green checkmark indicating the ping was delivered successfully.

If the ping failed:
- Check that port 8080 is accessible from the internet
- Check your server's firewall: `sudo ufw status`
- Check your cloud provider's firewall/security group settings

### Step 5.3 — Configure Jenkins to Accept Webhooks

Jenkins should be ready to receive webhooks automatically if the **GitHub Integration Plugin** is installed.

To verify:

1. Go to **Manage Jenkins** → **System**
2. Scroll to the **GitHub** section
3. Ensure **Manage hooks** is selected
4. If you see errors, add your GitHub server credentials:

   - Click **Add GitHub Server** → **Credentials** → Add your `github-token`

---

## Part 6: Test the Full CI/CD Flow

Now test the entire pipeline end-to-end:

### Step 6.1 — Make a Code Change

Make a small change to your NextJS app (edit the homepage text, for example):

```bash
# On your local machine or server
echo "console.log('Auto-deploy test');" >> app/page.js
```

### Step 6.2 — Commit and Push

```bash
git add .
git commit -m "Test auto-deploy pipeline"
git push origin main
```

### Step 6.3 — Watch the Automation

1. In Jenkins, your pipeline should trigger automatically within seconds of the push
2. Watch the console output as Jenkins:
   - Clones the latest code
   - Rebuilds Docker images
   - Deploys the updated containers
3. Refresh your browser at `http://YOUR_SERVER_IP` to see the change

---

## The Complete CI/CD Flow

```
Developer pushes code to GitHub
        ↓
GitHub webhook fires (HTTP POST to Jenkins)
        ↓
Jenkins pipeline triggers
        ↓
Clones the latest code
        ↓
Runs docker compose build
        ↓
Runs docker compose up -d
        ↓
Application updates live
        ↓
Cleanup removes old images
```

---

## Verification Checklist

Before moving on, confirm:

- [x] Jenkins can manually run the pipeline and it completes successfully
- [x] The `Jenkinsfile` is committed to your GitHub repository
- [x] GitHub webhook is configured and the test ping succeeded
- [x] Jenkins pipeline triggers automatically when you push code
- [x] The application updates after a Git push without manual intervention

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `docker: command not found` in Jenkins pipeline | Jenkins user not in docker group. Run `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| `Host key verification failed` when cloning | Jenkins cannot verify the GitHub host key. Run this once as the jenkins user: `sudo -u jenkins ssh-keyscan github.com >> ~jenkins/.ssh/known_hosts` |
| Webhook ping returns `301` or `Connection refused` | Jenkins may not be accessible from the internet. Check firewall (UFW) and cloud security group. For testing, you can use `ngrok` to expose port 8080 |
| Pipeline fails at `docker compose build` | Ensure the Jenkinsfile's working directory contains `docker-compose.yml`. The `git` step should clone into the workspace |
| `permission denied while trying to connect to the Docker daemon` | Jenkins user needs docker group membership and a restart |

---

**Next:** [Chapter 8 — Production Hardening & Backups](08-production-hardening-and-backups.md)
