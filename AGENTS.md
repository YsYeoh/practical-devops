# Agents.md

## What this repo is

Documentation-only training repository. No source code, no build system, no CI config, no tests, no lint/format/typecheck tooling.

The training content is in `chapters/` — 20 files teaching NextJS + MongoDB + Nginx deployment via Docker Compose and Jenkins on Ubuntu 22.04, plus a Kubernetes extension (Minikube, kubectl, Helm, Ingress, StatefulSets, autoscaling).

Legacy files: `practical_nextjs_docker_compose_jenkins_training_guide.md` (original 22-phase guide, superseded by chapters) and its `.pdf` / `.docx` exports.

## What this repo is NOT

- Not a runnable project — no `package.json`, `Dockerfile`, `docker-compose.yml`, `Jenkinsfile`, or any executable code
- Not a library or framework
- Not a multi-package monorepo

## Commands

None. There are no build, test, lint, format, or typecheck commands.

## Conventions

- The markdown guide is the source of truth. PDF/DOCX are generated exports.
- All infrastructure, pipeline, and deployment code described in the guide exists as inline markdown blocks only — never as standalone files.
- No instruction files (`.cursorrules`, `CLAUDE.md`, `opencode.json`, etc.) existed before this file.
