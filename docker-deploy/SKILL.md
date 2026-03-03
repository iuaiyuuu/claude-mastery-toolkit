---
name: docker-deploy
description: >
  Generate a production-ready DevOps deployment setup for deploying a Dockerized web application
  to a single server using GitHub Actions CI/CD, Docker Compose, and Nginx reverse proxy.
  Use this skill whenever the user asks to deploy a web app to a server, set up CI/CD with Docker,
  create a deployment pipeline, configure Nginx as a reverse proxy for a Docker container, set up
  GitHub Actions for SSH deployment, or asks for a "DevOps template", "deployment scaffold",
  "production deployment setup", or "deploy to VPS". Also trigger when the user mentions Docker +
  GitHub Actions together, asks about deploying to a single server (DigitalOcean, Hetzner, Linode,
  any VPS), or needs a reproducible IaC-style deployment without Kubernetes or cloud-managed services.
  Trigger even if the user says "deploy my app" without specifying Docker — if they have a web app
  and a server, this skill applies. Do NOT trigger for Kubernetes, AWS ECS, Google Cloud Run,
  Terraform cloud provisioning, or multi-server orchestration.
---

# Docker Deploy Skill

Generate a complete, production-ready DevOps deployment template for a Dockerized web application on a single server.

## When to use

This skill produces actionable deployment artifacts — no theory, no alternatives, no optional steps. It targets the most common indie/startup deployment pattern: a single server running Docker with Nginx reverse proxy, deployed via GitHub Actions over SSH.

## Default stack

- **Source control:** GitHub repository
- **Application:** Dockerized app (default port 3000, configurable)
- **CI/CD:** GitHub Actions (raw SSH — no third-party actions except actions/checkout)
- **Image registry:** GitHub Container Registry (ghcr.io)
- **Deployment:** SSH to single server, docker pull from GHCR
- **Reverse proxy:** Nginx (containerized via Docker Compose)
- **Secrets:** Environment variables only — never plaintext in files

## Output format

Generate ALL of the following sections in order. Do not skip any. Place clarifications as inline comments in code, not as prose explanations.

### 1. Architecture overview (max 5 lines of prose)

Describe the flow: push → GitHub Actions builds image → SSH deploy → Docker Compose up → Nginx proxies traffic. Keep it to 5 lines max.

### 2. Directory structure

Show as a tree. Use this as the canonical layout:

```
project-root/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── docker-compose.yml
├── docker-compose.prod.yml
├── Dockerfile
├── nginx/
│   └── default.conf
├── .env.example
└── README.md
```

### 3. Dockerfile

Read `references/dockerfile.md` for the template. Key requirements:
- Multi-stage build (builder + production)
- Non-root user
- Health check
- .dockerignore included

### 4. docker-compose.yml (and docker-compose.prod.yml)

Read `references/docker-compose.md` for the template. Key requirements:
- Health checks on all services
- Restart policy: `unless-stopped`
- Named volumes for persistence
- Environment variables via `.env` file
- Production override file for prod-specific config
- Nginx service with port 80/443 exposed
- App service NOT exposed to host (only via Nginx network)

### 5. Nginx configuration

Read `references/nginx.md` for the template. Key requirements:
- Reverse proxy to app container by service name
- Proxy headers (X-Real-IP, X-Forwarded-For, X-Forwarded-Proto)
- Gzip enabled
- Security headers
- Health check endpoint passthrough

### 6. GitHub Actions workflow

Read `references/github-actions.md` for the template. Key requirements:
- Trigger on push to `main`
- Build Docker image and push to GHCR
- SSH into server using raw `ssh`/`scp` commands (no third-party actions beyond actions/checkout)
- Pull image from GHCR on server
- Use GitHub Secrets for all sensitive values
- No plaintext secrets anywhere
- Cleanup SSH key material after deploy
- Includes deployment verification step

### 7. Deployment commands (server-side)

Read `references/server-setup.md` for the template. Key requirements:
- First-time server setup commands
- Separate Linux and Windows Server sections where needed
- Docker and Docker Compose installation
- Directory setup and permissions
- Firewall rules (ufw)

### 8. Verification steps

Provide concrete commands to verify:
- Container health: `docker compose ps`
- App response: `curl -f http://localhost:3000/health`
- Nginx proxy: `curl -f http://your-server-ip`
- Logs: `docker compose logs -f`

### 9. Common failure checklist

Read `references/troubleshooting.md` for the checklist.

## Customization points

When the user specifies these, adapt the template accordingly:
- **App port:** Default 3000, replace throughout
- **App language/framework:** Adjust Dockerfile base image and build steps
- **Domain name:** Add to Nginx server_name and suggest certbot
- **Multiple services:** Add to docker-compose with appropriate networking
- **Database:** Add postgres/mysql service with volume, health check, and dependency

If the user doesn't specify, use the defaults and note where they can customize.

## Critical rules

1. Never put secrets in files committed to git. Always use environment variables or GitHub Secrets.
2. Every service must have a health check.
3. Every service must have `restart: unless-stopped`.
4. The app container must NOT expose ports to the host — only Nginx is exposed.
5. All commands must work on a fresh server with only Docker installed.
6. Include `.env.example` with placeholder values, never real secrets.
