# practical-devops

Practical DevOps training guide covering deployment of a fullstack NextJS app with MongoDB and Nginx via Docker Compose, automated with Jenkins CI/CD on Ubuntu 22.04, with a Kubernetes extension (Minikube, Helm, Ingress, StatefulSets).

## Contents

- `chapters/` — 20 modular markdown chapters (the source of truth):
  - Chapters 1–12: Docker Compose, Jenkins, production deployment
  - Chapters 13–20: Kubernetes extension (Minikube, kubectl, Helm, Ingress, StatefulSets, HPA)
- `practical_nextjs_docker_compose_jenkins_training_guide.md` — legacy monolithic guide (superseded by chapters)
- `.pdf` / `.docx` — generated exports of the legacy guide

## Chapter Overview

| # | Title |
|---|-------|
| 1 | Introduction & Prerequisites |
| 2 | Server Setup & Docker |
| 3 | Project Structure & Dockerfile |
| 4 | Docker Compose & Networking |
| 5 | First Deployment & Testing |
| 6 | Jenkins Installation & Setup |
| 7 | GitHub Integration & Pipeline |
| 8 | Production Hardening & Backups |
| 9 | Full Ecosystem Migration |
| 10 | Advanced Architecture & Monitoring |
| 11 | Secrets Management & Zero-Downtime |
| 12 | Final Practice & Learning Path |
| 13 | Kubernetes Introduction & Core Concepts |
| 14 | Setting Up a Local Kubernetes Cluster |
| 15 | Pods, Deployments & Services |
| 16 | ConfigMaps, Secrets & Configuration |
| 17 | StatefulSets & Persistent Storage |
| 18 | Kubernetes Ingress & Networking |
| 19 | Helm Package Manager |
| 20 | Production Practices, CI/CD & Next Steps |

## What You Will Learn

- Set up an Ubuntu server from scratch with Docker
- Write a multi-stage Dockerfile for a NextJS app
- Configure Nginx as a reverse proxy inside Docker
- Define and run a multi-service stack with Docker Compose
- Install and configure Jenkins for CI/CD
- Connect Jenkins to GitHub with webhooks for automatic deployment
- Harden the server with HTTPS and a firewall
- Back up and restore MongoDB
- Migrate the entire stack to a new server
- Monitor with Prometheus, Grafana, and Loki
- Manage secrets and implement blue-green zero-downtime deployment
- Deploy and manage apps on Kubernetes with Minikube, Helm, and Ingress

## What You Will Understand

After completing this training, you will understand:

- **Docker lifecycle** — build, run, stop, prune, multi-stage builds
- **Container orchestration** — Docker Compose services, volumes, networks, internal DNS
- **Reverse proxy** — Nginx routing, SSL termination
- **CI/CD pipelines** — Jenkinsfile as code, GitHub webhooks, automated builds & deployments
- **Production hardening** — HTTPS with Let's Encrypt, UFW firewall, MongoDB backup/restore
- **Infrastructure as Code** — repeatable, version-controlled infrastructure
- **Observability** — metrics, logs, and monitoring dashboards
- **Zero-downtime deployment** — blue-green strategy with health checks
- **Kubernetes fundamentals** — Pods, Deployments, Services, ConfigMaps, Secrets, StatefulSets, Ingress, Helm

## Prerequisites

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Storage | 40 GB SSD | 80 GB SSD |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

- An Ubuntu 22.04 server (physical, VM, or cloud instance) with SSH access
- A GitHub account and repository
- A domain name (optional, needed for HTTPS in production)
- Basic familiarity with the Linux command line and Git
- Node.js on your local machine (to create a NextJS app)

### Software Versions

| Software | Version |
|---|---|
| Docker Engine | Latest stable |
| Docker Compose Plugin | Latest stable |
| Node.js | 20 (Alpine) |
| MongoDB | 7 |
| Nginx | Latest |
| Jenkins | Latest LTS |
| Java | OpenJDK 17 |
| Ubuntu | 22.04 LTS |
