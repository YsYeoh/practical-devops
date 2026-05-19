# Chapter 9: Full Ecosystem Migration

This chapter demonstrates the real power of Infrastructure as Code (IaC): migrating the entire application stack from one server to another with minimal effort.

---

## The Scenario

```
Current Server (Server A)
    ↓
  Move to
    ↓
New Server (Server B)
```

You need to move everything:
- NextJS application
- MongoDB database
- Nginx configuration
- Docker Compose setup
- Jenkins CI/CD pipeline
- Environment variables

Without containers, this would mean:
- Manually installing Node.js, MongoDB, Nginx, Jenkins on the new server
- Copying configurations one by one
- Worrying about version mismatches
- Debugging environment differences

With Docker Compose and IaC, migration becomes a repeatable, scriptable process.

---

## Step 1: Prepare the New Server (Server B)

SSH into the new server and install Docker (exactly as you did in Chapter 2):

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install prerequisites
sudo apt install -y curl git unzip ca-certificates gnupg lsb-release

# Add Docker's GPG key and repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Add your user to the docker group
sudo usermod -aG docker $USER

# Reconnect SSH before running Docker commands
exit
```

Then reconnect to Server B.

---

## Step 2: Transfer the Project Source

On Server B, clone the same GitHub repository:

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git ~/my-project
cd ~/my-project
```

This brings over:
- `Dockerfile`
- `docker-compose.yml`
- `nginx/default.conf`
- `Jenkinsfile`
- Application source code

Everything defined in code is now on the new server.

---

## Step 3: Transfer Environment Variables

The `.env` file is (correctly) not in Git, so you need to transfer it separately.

From your local machine or Server A:

```bash
scp ~/my-project/.env user@NEW_SERVER_IP:~/my-project/.env
```

If you do not have the `.env` file backed up, simply recreate it on Server B:

```bash
nano ~/my-project/.env
```

```env
MONGODB_URI=mongodb://admin:password123@mongodb:27017
NODE_ENV=production
PORT=3000
```

---

## Step 4: Transfer the MongoDB Backup

### Step 4.1 — Create a Fresh Backup on Server A

```bash
# On Server A
docker exec mongodb mongodump \
  --username admin \
  --password password123 \
  --archive=/tmp/dump.gz \
  --gzip

docker cp mongodb:/tmp/dump.gz ~/dump.gz
```

### Step 4.2 — Copy the Backup to Server B

From your local machine or Server A:

```bash
scp ~/dump.gz user@NEW_SERVER_IP:~/dump.gz
```

---

## Step 5: Start the Stack on Server B

```bash
cd ~/my-project
docker compose up -d
```

Docker Compose will:
1. Pull the `mongo:7` and `nginx:latest` images
2. Build the NextJS image using the Dockerfile
3. Create the `app-network`
4. Create the `mongodb_data` volume
5. Start all three containers

---

## Step 6: Restore the Database on Server B

```bash
# Copy backup into the MongoDB container
docker cp ~/dump.gz mongodb:/dump.gz

# Restore the database
docker exec mongodb mongorestore \
  --username admin \
  --password password123 \
  --archive=/dump.gz \
  --gzip

# Clean up
rm ~/dump.gz
```

---

## Step 7: Verify the Migration

1. Check that all containers are running:

   ```bash
   docker ps
   ```

2. Check that the application is accessible:

   ```bash
   curl http://localhost
   ```

3. Check that the database has data:

   ```bash
   docker exec mongodb mongosh \
     --username admin \
     --password password123 \
     --eval "show dbs"
   ```

4. From your browser, visit `http://NEW_SERVER_IP`

---

## Step 8: Set Up Jenkins (Optional)

If you want CI/CD on the new server, repeat the Jenkins installation (Chapter 6) and pipeline setup (Chapter 7).

Since the `Jenkinsfile` is in Git, you only need to:

1. Install Jenkins
2. Create a new pipeline job pointing to the same GitHub repository
3. Configure the GitHub webhook to point to the new Jenkins server

---

## Migration Checklist

Use this checklist when migrating to ensure nothing is missed:

- [ ] Server B has Docker Engine and Compose installed
- [ ] Project source cloned from GitHub
- [ ] `.env` file transferred or recreated
- [ ] MongoDB backup created on Server A
- [ ] Backup transferred to Server B
- [ ] `docker compose up -d` starts successfully on Server B
- [ ] Database restored from backup
- [ ] Application accessible at `http://NEW_SERVER_IP`
- [ ] DNS updated to point domain to new IP (if applicable)
- [ ] Jenkins reinstalled and pipeline configured (if needed)
- [ ] Old server decommissioned (after validation)

---

## Key Lesson

Notice what you did **not** do on the new server:

- Did not manually install Node.js
- Did not manually install MongoDB
- Did not manually configure Nginx
- Did not manually set up runtime dependencies
- Did not worry about version compatibility

Everything came from:
- **Dockerfile** — defines the application runtime
- **docker-compose.yml** — defines the infrastructure
- **GitHub repository** — stores everything as code

This is the essence of **Infrastructure as Code**: your infrastructure becomes portable, version-controlled, and reproducible.

---

## Verification Checklist

Before moving on, confirm:

- [x] Server B has Docker installed and running
- [x] The application builds and runs on Server B without changes
- [x] MongoDB data has been restored and is accessible
- [x] The application is accessible via the browser on the new server
- [x] You understand what made this migration easy (IaC, Docker, Compose)

---

**Next:** [Chapter 10 — Advanced Architecture & Monitoring](10-advanced-architecture-and-monitoring.md)
