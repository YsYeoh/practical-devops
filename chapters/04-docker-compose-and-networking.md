# Chapter 4: Docker Compose & Networking

In this chapter, you will set up Nginx as a reverse proxy and define the full application stack in a `docker-compose.yml` file.

---

## Step 1: Create the Nginx Configuration

Create `nginx/default.conf`:

```bash
nano nginx/default.conf
```

Paste the following:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://nextjs:3000;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### What This Config Does

| Directive | Purpose |
|---|---|
| `listen 80` | Accept HTTP traffic on port 80 |
| `proxy_pass http://nextjs:3000` | Forward requests to the `nextjs` container on port 3000 |
| `proxy_http_version 1.1` | Required for WebSocket support (NextJS uses WebSocket for hot reloading) |
| `proxy_set_header Upgrade ...` | Pass WebSocket upgrade headers |
| `proxy_set_header Host $host` | Preserve the original Host header |
| `proxy_cache_bypass` | Bypass cache for WebSocket connections |

### Key Insight: Docker DNS

Notice the URL `http://nextjs:3000`. This is **not** `localhost` — it uses Docker Compose's internal DNS. Compose automatically creates a network where each service name (like `nextjs`) resolves to that container's IP address. This means containers can find each other by name, without hardcoding IPs.

---

## Step 2: Create `docker-compose.yml`

Create the file in the project root:

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
services:
  nextjs:
    build:
      context: .
    container_name: nextjs-app
    restart: always
    env_file:
      - .env
    depends_on:
      - mongodb
    networks:
      - app-network

  mongodb:
    image: mongo:7
    container_name: mongodb
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
    volumes:
      - mongodb_data:/data/db
    networks:
      - app-network

  nginx:
    image: nginx:latest
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - nextjs
    networks:
      - app-network

volumes:
  mongodb_data:

networks:
  app-network:
```

---

## Understanding Each Service

### `nextjs` Service

```yaml
nextjs:
  build:
    context: .
  container_name: nextjs-app
  restart: always
  env_file:
    - .env
  depends_on:
    - mongodb
  networks:
    - app-network
```

- **`build.context: .`** — Builds the Dockerfile in the current directory
- **`env_file`** — Loads environment variables from `.env`
- **`depends_on`** — Ensures MongoDB starts before NextJS (does not wait for it to be ready, just for the container to start)
- **`networks`** — Attaches to `app-network` for internal communication

### `mongodb` Service

```yaml
mongodb:
  image: mongo:7
  container_name: mongodb
  restart: always
  ports:
    - "27017:27017"
  environment:
    MONGO_INITDB_ROOT_USERNAME: admin
    MONGO_INITDB_ROOT_PASSWORD: password123
  volumes:
    - mongodb_data:/data/db
  networks:
    - app-network
```

- **`image: mongo:7`** — Pulls the official MongoDB 7 image (no Dockerfile needed)
- **`ports: "27017:27017"`** — Exposes MongoDB on the host port 27017 (useful for external tools like MongoDB Compass)
- **`volumes: mongodb_data:/data/db`** — Persists database data in a named volume. Without this, all data is lost when the container stops

### `nginx` Service

```yaml
nginx:
  image: nginx:latest
  container_name: nginx
  restart: always
  ports:
    - "80:80"
  volumes:
    - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - nextjs
  networks:
    - app-network
```

- Mounts your local `nginx/default.conf` into the container, overwriting the default Nginx config
- Depends on `nextjs` (Nginx has nothing to proxy until the app is running)

---

## Understanding Volumes

```yaml
volumes:
  mongodb_data:
```

**What this does:** Declares a named Docker volume called `mongodb_data`. Docker stores this volume in `/var/lib/docker/volumes/` on the host. The data persists across container restarts, rebuilds, and even survives `docker compose down`.

**Why it matters:** Without a volume:
- If the MongoDB container is deleted (`docker compose down` or `docker rm`), all database data is gone forever
- With the volume, the data survives container restarts and upgrades

---

## Understanding Networks

```yaml
networks:
  app-network:
```

All three services are attached to `app-network`, which means:

- They can communicate using service names (`nextjs`, `mongodb`, `nginx`) instead of IP addresses
- The network is isolated — only services on this network can reach each other
- Services not on this network cannot access these containers (unless ports are explicitly published)

---

## Step 3: Create `.env` File

Create the environment variables file:

```bash
nano .env
```

Paste:

```env
MONGODB_URI=mongodb://admin:password123@mongodb:27017
NODE_ENV=production
PORT=3000
```

**Important notes about `.env`:**

- The `MONGODB_URI` uses `mongodb` as the hostname — this is the Docker Compose service name, not `localhost`
- Never commit `.env` to Git. Add it to `.gitignore`:
  ```bash
  echo ".env" >> .gitignore
  ```
- In a real production environment, use Docker Secrets or a vault instead of plain-text `.env` (covered in Chapter 11)

---

## Verification Checklist

Before moving on, confirm:

- [x] `nginx/default.conf` exists with the proxy configuration
- [x] `docker-compose.yml` exists with all three services
- [x] `.env` exists (and is listed in `.gitignore`)
- [x] You understand how Docker Compose service names resolve as hostnames

---

## Common Pitfalls

| Problem | Solution |
|---|---|
| Nginx cannot connect to `nextjs:3000` | Make sure both services are on the same Docker network (`app-network`) |
| MongoDB connection refused | The `depends_on` only waits for the container to start, not for MongoDB to be ready. Your app should implement a retry mechanism on startup |
| Port `80` already in use | Check if another web server (Apache, Nginx) is running: `sudo lsof -i :80`. Stop it with `sudo systemctl stop apache2` |
| Changes to `default.conf` not reflected in Nginx | Run `docker compose restart nginx` to reload the config |

---

**Next:** [Chapter 5 — First Deployment & Testing](05-first-deployment-and-testing.md)
