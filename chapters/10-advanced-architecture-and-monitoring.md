# Chapter 10: Advanced Architecture & Monitoring

This chapter introduces a production-grade, AI-ready microservice architecture and a complete monitoring stack.

---

## Part 1: Advanced Production Architecture

The basic three-service stack (NextJS + MongoDB + Nginx) is a starting point. A real production system, especially one involving AI, requires additional components.

### Target Architecture

```
                    Internet
                        ↓
                    Nginx Proxy
                        ↓
        ┌─────────────────────────────┐
        │         NextJS Frontend      │
        └─────────────────────────────┘
                        ↓
        ┌─────────────────────────────┐
        │        Backend API          │
        │   (Express / FastAPI)       │
        └─────────────────────────────┘
            ↓         ↓         ↓
         MongoDB    Redis     Qdrant
                         ↓
                    Queue Worker
                         ↓
                     AI Service
                         ↓
                  Ollama / LLM

─────────────────────────────────────

Monitoring Layer: Prometheus + Grafana + Loki
CI/CD Layer: GitHub → Jenkins → Docker Compose
```

### Service Responsibilities

| Service | Role |
|---|---|
| **Frontend** (NextJS) | UI, authentication, dashboard, real-time updates, file upload |
| **Backend API** | Business logic, JWT auth, database access, API gateway, queue management |
| **Redis** | Async job queue for AI processing, notifications, image processing |
| **Worker** | Processes queued jobs (AI inference, document parsing, etc.) |
| **AI Service** | Embedding generation, LLM inference, RAG, prompt orchestration |
| **Qdrant** | Vector database for embeddings, semantic search, AI memory |

### Why a Queue (Redis)?

Without a queue:

```
User request → AI processing (30+ seconds) → Response
User waits, browser times out
```

With a queue:

```
User request → Redis Queue → "Job submitted" (instant response)
                               ↓
                          Worker processes in background
                               ↓
                          Frontend polls/subscribes for result
```

The queue pattern is essential whenever you have:
- Long-running operations (AI inference > 5 seconds)
- Unpredictable processing times
- Need to decouple request from response

### Advanced Project Structure

```
ai-platform/
├── frontend/             # NextJS
├── backend/              # Express / FastAPI / NestJS
├── worker/               # Queue worker
├── ai-service/           # AI processing (Python FastAPI)
├── nginx/                # Reverse proxy config
├── monitoring/
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── grafana/
│   │   └── provisioning/
│   └── loki/
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env
├── .env.production
├── secrets/
└── Jenkinsfile
```

### Advanced Docker Compose

```yaml
services:
  frontend:
    build: ./frontend
    restart: always
    depends_on:
      - backend
    networks:
      - app-network

  backend:
    build: ./backend
    restart: always
    env_file:
      - .env
    depends_on:
      - mongodb
      - redis
      - qdrant
    networks:
      - app-network

  worker:
    build: ./worker
    restart: always
    depends_on:
      - redis
      - ai-service
    networks:
      - app-network

  ai-service:
    build: ./ai-service
    restart: always
    networks:
      - app-network

  mongodb:
    image: mongo:7
    restart: always
    volumes:
      - mongodb_data:/data/db
    networks:
      - app-network

  redis:
    image: redis:7
    restart: always
    networks:
      - app-network

  qdrant:
    image: qdrant/qdrant
    restart: always
    volumes:
      - qdrant_data:/qdrant/storage
    networks:
      - app-network

  nginx:
    image: nginx:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend
    networks:
      - app-network

volumes:
  mongodb_data:
  qdrant_data:

networks:
  app-network:
```

---

## Part 2: Monitoring Stack

Production systems without monitoring are blind. You need to know:
- Is the server healthy?
- Are containers running?
- Is the application responding?
- Is there a memory leak?
- How many requests are failing?

### Monitoring Components

| Component | Purpose | Port |
|---|---|---|
| **Prometheus** | Collects and stores metrics from all services | 9090 |
| **Grafana** | Visualizes metrics in dashboards | 3001 |
| **Loki** | Aggregates and searches logs | 3100 |
| **cAdvisor** | Exposes container-level metrics (CPU, memory, network) | 8081 |
| **Node Exporter** | Exposes server-level metrics | 9100 |

### Step 1 — Add Monitoring Services to Docker Compose

Append these services to your `docker-compose.yml`:

```yaml
  prometheus:
    image: prom/prometheus
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - app-network

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - app-network

  loki:
    image: grafana/loki
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki
    networks:
      - app-network

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8081:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - app-network

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      - app-network

volumes:
  mongodb_data:
  qdrant_data:
  prometheus_data:
  grafana_data:
  loki_data:
```

### Step 2 — Configure Prometheus

Create the Prometheus config file:

```bash
mkdir -p monitoring/prometheus
nano monitoring/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

### Step 3 — Start the Monitoring Stack

```bash
docker compose up -d
```

### Step 4 — Access Monitoring Tools

| Tool | URL |
|---|---|
| Prometheus | `http://YOUR_SERVER_IP:9090` |
| Grafana | `http://YOUR_SERVER_IP:3001` |
| cAdvisor | `http://YOUR_SERVER_IP:8081` |
| Node Exporter | `http://YOUR_SERVER_IP:9100` |

### Step 5 — Configure Grafana

1. Visit `http://YOUR_SERVER_IP:3001`
2. Login with default credentials: `admin` / `admin`
3. Change the password when prompted
4. Add Prometheus as a data source:

   - Go to **Configuration** → **Data Sources** → **Add data source**
   - Select **Prometheus**
   - Set URL to `http://prometheus:9090`
   - Click **Save & Test**

5. Import a dashboard:

   - Go to **+** → **Import**
   - Enter dashboard ID `1860` (Node Exporter Full) and click **Load**
   - Select the Prometheus data source
   - Click **Import**

### Step 6 — What to Monitor

| Category | Key Metrics |
|---|---|
| **Server** | CPU usage, RAM usage, disk usage, network traffic |
| **Containers** | Restart count, memory usage, CPU usage, health status |
| **Application** | API latency (p50/p95/p99), request count, error rate, active connections |
| **AI Stack** | Token usage, inference latency, embedding latency, queue size, failure rate |
| **Database** | Connection count, query latency, disk usage |

### Recommended Grafana Dashboards

| Dashboard ID | Description |
|---|---|
| `1860` | Node Exporter Full (server metrics) |
| `14282` | cAdvisor Exporter (container metrics) |
| `13659` | Loki Logs (log aggregation) |

Find more at [grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)

---

## Verification Checklist

Before moving on, confirm:

- [x] You understand the advanced microservice architecture
- [x] Prometheus is accessible at `http://YOUR_SERVER_IP:9090`
- [x] Grafana is accessible at `http://YOUR_SERVER_IP:3001`
- [x] Grafana can query Prometheus as a data source
- [x] You can see server metrics in a dashboard (CPU, memory, disk)
- [x] cAdvisor shows container metrics at `http://YOUR_SERVER_IP:8081`

---

**Next:** [Chapter 11 — Secrets Management & Zero Downtime Deployment](11-secrets-management-and-zero-downtime.md)
