# Chapter 11: Secrets Management & Zero Downtime Deployment

This chapter covers two critical production practices: keeping secrets out of your codebase and deploying without downtime.

---

## Part 1: Docker Secrets Management

### The Problem

Hardcoding secrets is dangerous:

```env
MONGODB_PASSWORD=password123
```

What is wrong with this:
- The `.env` file can be accidentally committed to Git
- Anyone with access to the server can read it
- No audit trail for who accessed what
- Hard to rotate credentials

### Docker Secrets Concept

Docker provides a built-in secrets mechanism:

```
Secret stored securely on host
         ↓
Mounted as file inside container at /run/secrets/
         ↓
Application reads the secret from the file
```

Secrets are:
- Never stored in the image
- Never visible in `docker inspect`
- Only mounted into containers that explicitly request them

### Step 1 — Create Secret Files

```bash
mkdir -p secrets
```

Create secret files (one per secret):

```bash
echo -n "MyStr0ng!Passw0rd" > secrets/mongodb_password.txt
echo -n "your-jwt-secret-here" > secrets/jwt_secret.txt
echo -n "sk-your-openai-key" > secrets/openai_api_key.txt
```

> **Security note:** The `-n` flag prevents a trailing newline from being added to the secret file. A newline would become part of the secret value and cause authentication failures.

### Step 2 — Add Secrets to Docker Compose

Update your `docker-compose.yml`:

```yaml
services:
  backend:
    build: ./backend
    restart: always
    env_file:
      - .env
    depends_on:
      - mongodb
      - redis
    secrets:
      - mongodb_password
      - jwt_secret
    networks:
      - app-network

  # ... other services ...

secrets:
  mongodb_password:
    file: ./secrets/mongodb_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
```

### Step 3 — Read Secrets in Your Application

In your Node.js/NextJS application:

```javascript
const fs = require('fs');

const mongodbPassword = fs.readFileSync('/run/secrets/mongodb_password', 'utf8').trim();
const jwtSecret = fs.readFileSync('/run/secrets/jwt_secret', 'utf8').trim();

// Use the secret to construct the MongoDB URI
const MONGODB_URI = `mongodb://admin:${mongodbPassword}@mongodb:27017`;
```

In Python (for AI service):

```python
with open('/run/secrets/openai_api_key', 'r') as f:
    openai_api_key = f.read().strip()
```

### Important Security Rules

| Rule | Why |
|---|---|
| Never commit secrets to Git | Add `secrets/` to `.gitignore` |
| Never expose `.env` publicly | Rotate immediately if leaked |
| Never hardcode API keys | Use Docker Secrets or a vault |
| Restrict file permissions | `chmod 600 secrets/*` |
| Use different passwords per environment | Dev, staging, production should each have unique secrets |

### Beyond Docker Secrets

For production, learn these more advanced secret management tools:

| Tool | Description |
|---|---|
| **HashiCorp Vault** | Enterprise-grade secrets management with dynamic secrets |
| **AWS Secrets Manager** | Managed secrets with automatic rotation |
| **Doppler** | Cloud-native secret management platform |
| **Kubernetes Secrets** | Built-in secrets for Kubernetes |

---

## Part 2: Zero Downtime Deployment (Blue-Green)

### The Problem

Normal deployment causes downtime:

```
Stop old container → (downtime here) → Start new container
```

Even a few seconds of downtime can cost money and frustrate users.

### Blue-Green Deployment Strategy

```
Current: Blue (v1)   New: Green (v2)
```

Flow:

```
1. Deploy Green (v2) alongside Blue (v1)
2. Wait for Green's health check to pass
3. Switch Nginx traffic from Blue to Green
4. Remove Blue

No downtime because Blue handles traffic until Green is ready.
```

### Simple Blue-Green Implementation

#### Step 1 — Define Two App Services

In `docker-compose.yml`:

```yaml
services:
  nextjs-blue:
    build:
      context: .
      args:
        - VERSION=blue
    container_name: nextjs-blue
    restart: always
    env_file:
      - .env
    networks:
      - app-network
    profiles:
      - blue

  nextjs-green:
    build:
      context: .
      args:
        - VERSION=green
    container_name: nextjs-green
    restart: always
    env_file:
      - .env
    networks:
      - app-network
    profiles:
      - green
```

#### Step 2 — Nginx with Active/Standby Upstream

```nginx
upstream app {
    server nextjs-blue:3000 weight=1;
    # server nextjs-green:3000 weight=1;  # Uncomment to switch
}

server {
    listen 80;

    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

#### Step 3 — Manual Blue-Green Switch

Deploy Green:

```bash
docker compose --profile green up -d --build
```

Wait for the health check to pass, then switch Nginx traffic by updating `default.conf` and reloading:

```bash
docker compose restart nginx
```

Remove the old (Blue) deployment:

```bash
docker compose --profile blue down
```

### Health Check Configuration

Docker Compose supports health checks that verify a container is actually responding before considering it healthy:

```yaml
services:
  nextjs:
    build: .
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
```

| Parameter | Purpose |
|---|---|
| `test` | Command that runs inside the container to check health |
| `interval` | How often to check |
| `timeout` | How long to wait for the check to complete |
| `retries` | Consecutive failures before marking unhealthy |
| `start_period` | Grace period before health checks begin |

### Zero Downtime Jenkins Pipeline

```groovy
pipeline {
    agent any

    environment {
        COMPOSE_PROJECT_NAME = "ai-platform"
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'YOUR_GITHUB_REPO'
            }
        }

        stage('Build Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'docker compose run backend npm test'
            }
        }

        stage('Deploy Green') {
            steps {
                sh 'docker compose --profile green up -d --build'
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    sleep 10
                    curl -f http://localhost:3000/health || exit 1
                '''
            }
        }

        stage('Switch Traffic') {
            steps {
                sh '''
                    # Update Nginx upstream to point to green
                    sed -i 's/server nextjs-blue:3000/server nextjs-green:3000/' nginx/default.conf
                    docker compose restart nginx
                '''
            }
        }

        stage('Remove Blue') {
            steps {
                sh 'docker compose --profile blue down'
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

---

## Verification Checklist

Before moving on, confirm:

- [x] You understand why `.env` files should not be committed to Git
- [x] You can create Docker secrets and mount them into a container
- [x] Your application can read secrets from `/run/secrets/`
- [x] `secrets/` is in your `.gitignore`
- [x] You understand why blue-green deployment avoids downtime
- [x] You can perform a manual blue-green switch
- [x] Health checks are configured for your application

---

**Next:** [Chapter 12 — Final Practice & Learning Path](12-final-practice-and-learning-path.md)
