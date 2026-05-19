# Chapter 5: First Deployment & Testing

In this chapter, you will build and start your Docker stack for the first time, verify everything is running, and learn the essential Docker Compose commands for day-to-day operations.

---

## Step 1: Build the Docker Images

From the project root (`~/my-project`):

```bash
docker compose build
```

**What this does:** Reads `docker-compose.yml`, finds the `nextjs` service (which has a `build` directive), and builds its Docker image using the `Dockerfile`. For `mongodb` and `nginx` (which use `image:` instead of `build:`), it pulls those images from Docker Hub.

**Expected output:** You will see:
- Layer-by-layer output for the NextJS build (npm install, npm run build, etc.)
- `mongodb` and `nginx` layers being pulled from Docker Hub

> **First build is slow.** Subsequent builds will be much faster because Docker caches layers. Only layers that change (typically the `COPY . .` and later steps) are rebuilt.

---

## Step 2: Start the Stack

```bash
docker compose up -d
```

**What this does:** Creates and starts all containers in detached mode (`-d` means the containers run in the background, and your terminal is freed up).

Docker Compose will:
1. Create the `app-network` if it does not exist
2. Create the `mongodb_data` volume if it does not exist
3. Start `mongodb` first (due to `depends_on`)
4. Start `nextjs`
5. Start `nginx`

---

## Step 3: Verify Running Containers

```bash
docker ps
```

Expected output (three containers running):

```
CONTAINER ID   IMAGE              STATUS         PORTS
abc123   nginx:latest         Up 2 minutes   0.0.0.0:80->80/tcp
def456   my-project-nextjs    Up 2 minutes   3000/tcp
ghi789   mongo:7              Up 2 minutes   0.0.0.0:27017->27017/tcp
```

> Container names are `nginx`, `nextjs-app`, and `mongodb` (as specified in `docker-compose.yml`).

---

## Step 4: Test the Application

Open a web browser and navigate to:

```
http://YOUR_SERVER_IP
```

Replace `YOUR_SERVER_IP` with your server's IP address. You should see your NextJS application's homepage.

If you do not see the page, wait 10-15 seconds and try again — the NextJS server may still be initializing.

---

## Step 5: View Logs

To see what is happening inside the containers:

```bash
docker compose logs -f
```

**What this does:** Shows real-time logs from all services. `-f` (follow) keeps the log stream open so you see new entries as they appear.

**To view logs for a single service:**

```bash
docker compose logs -f nextjs
```

Press `Ctrl+C` to exit the log viewer.

---

## Step 6: Restart Services

To restart a specific service without stopping the others:

```bash
docker compose restart nginx
```

To restart all services:

```bash
docker compose restart
```

---

## Essential Docker Commands Reference

| Command | What It Does |
|---|---|
| `docker compose build` | Build (or rebuild) images |
| `docker compose up -d` | Start all containers in background |
| `docker compose down` | Stop and remove all containers, networks |
| `docker compose down -v` | Same as above, **plus** remove volumes (**warning: deletes database data**) |
| `docker compose restart` | Restart all services |
| `docker compose restart <service>` | Restart a single service |
| `docker compose logs -f` | View real-time logs from all services |
| `docker compose logs -f <service>` | View logs for a specific service |
| `docker compose ps` | List running containers (like `docker ps`) |
| `docker compose up -d --build` | Rebuild images and start fresh (useful after code changes) |
| `docker compose exec <service> <command>` | Run a command inside a running container (e.g., `docker compose exec mongodb mongosh`) |

---

## Step 7: Make a Code Change and Redeploy

To see the CI/CD concept in action (before Jenkins is set up), make a change manually:

1. Edit a page in your NextJS app (e.g., `app/page.js`)
2. Rebuild and restart:

```bash
docker compose up -d --build
```

**What this does:** Rebuilds the `nextjs` image with your changes and replaces the running container — all without touching MongoDB or Nginx.

---

## Step 8: Stop the Stack

When you are finished:

```bash
docker compose down
```

This stops and removes all containers and the `app-network`. The `mongodb_data` volume is **not** removed (unless you use `--volumes`), so your database data is preserved for next time.

---

## Verification Checklist

Before moving on, confirm:

- [x] `docker compose build` completes without errors
- [x] `docker compose up -d` starts all three containers
- [x] `docker ps` shows all three services running
- [x] Your NextJS app is accessible at `http://YOUR_SERVER_IP`
- [x] You can view logs with `docker compose logs -f`
- [x] You can rebuild and restart after code changes

---

## Troubleshooting

| Problem | Likely Cause | Solution |
|---|---|---|
| `port is already allocated` | Port 80 or 27017 is in use | Run `sudo lsof -i :80` to find the process. Stop Apache/Nginx if present |
| `Cannot connect to MongoDB` | NextJS started before MongoDB was ready | Add a retry/wait script in your NextJS app, or use a tool like `wait-for-it.sh` |
| `npm ERR!` during build | Missing or incorrect `package.json` | Check that `package.json` exists and has all required dependencies |
| Blank page in browser | NextJS build error or missing route | Check logs: `docker compose logs nextjs` |

---

**Next:** [Chapter 6 — Jenkins Installation & Setup](06-jenkins-installation-and-setup.md)
