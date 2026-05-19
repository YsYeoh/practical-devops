# Chapter 1: Introduction & Prerequisites

## What You Will Learn

This training will teach you to deploy a fullstack NextJS application with MongoDB and Nginx using Docker Compose, then automate the entire workflow with Jenkins CI/CD — all on Ubuntu 22.04.

## Final Architecture

```
GitHub Repository
        ↓
Git Push
        ↓
Jenkins Pipeline
        ↓
Build Docker Images
        ↓
Deploy with Docker Compose
        ↓
Ubuntu Server
 ├── Nginx (reverse proxy, port 80)
 ├── NextJS App (port 3000)
 ├── MongoDB (port 27017)
 └── Docker Network (internal)
```

## By the End of This Training

You will be able to:

1. Set up an Ubuntu server from scratch with Docker
2. Write a multi-stage Dockerfile for a NextJS app
3. Configure Nginx as a reverse proxy inside Docker
4. Define and run a multi-service stack with Docker Compose
5. Install and configure Jenkins for CI/CD
6. Connect Jenkins to GitHub with webhooks for automatic deployment
7. Harden the server with HTTPS and a firewall
8. Back up and restore MongoDB
9. Migrate the entire stack to a new server
10. Understand advanced topics: monitoring, secrets management, zero-downtime deployment

## Prerequisites

### Server Requirements

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Storage | 40 GB SSD | 80 GB SSD |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

### What You Need Before Starting

- An Ubuntu 22.04 server (physical, VM, or cloud instance) with SSH access
- A GitHub account and repository for your project
- A domain name (optional, needed for HTTPS in production)
- Basic familiarity with the Linux command line and Git
- Node.js installed on your local machine (to create a NextJS app, if you do not already have one)

### Software Versions Used in This Guide

| Software | Version |
|---|---|
| Docker Engine | Latest stable (via official repo) |
| Docker Compose Plugin | Latest stable |
| Node.js | 20 (Alpine) |
| MongoDB | 7 |
| Nginx | Latest |
| Jenkins | Latest stable (LTS) |
| Java | OpenJDK 17 |
| Ubuntu | 22.04 LTS |

## How to Use This Guide

Each chapter builds on the previous one. Follow them in order. Every step includes:

1. A clear instruction of what to do
2. The exact command(s) to run
3. An explanation of what the command does
4. A verification step to confirm it worked

## Project Overview

The project you will build:

```
my-project/
├── app/                  # NextJS application source
├── nginx/
│   └── default.conf      # Nginx reverse proxy config
├── docker-compose.yml    # Multi-service orchestration
├── Dockerfile            # Multi-stage build for NextJS
├── .env                  # Environment variables
└── Jenkinsfile           # CI/CD pipeline definition
```

---

**Next:** [Chapter 2 — Server Setup & Docker Installation](02-server-setup-and-docker.md)
