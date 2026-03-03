# Dockerfile Reference

## Standard multi-stage Dockerfile for a Node.js web application

Adapt the base image and build steps for the user's framework. The structure (multi-stage, non-root user, health check) stays the same regardless of language.

## Template

```dockerfile
# ---- Build stage ----
FROM node:20-alpine AS builder
WORKDIR /app

# Install dependencies first (layer caching)
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source and build
COPY . .
RUN npm run build || true
# "|| true" allows projects without a build step to pass

# ---- Production stage ----
FROM node:20-alpine AS production
WORKDIR /app

# Security: run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy built artifacts and production deps from builder
COPY --from=builder --chown=appuser:appgroup /app .

USER appuser

EXPOSE 3000

# Health check — adjust endpoint and interval as needed
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

## .dockerignore

Always generate alongside the Dockerfile:

```
node_modules
npm-debug.log
.git
.github
.env
.env.*
docker-compose*.yml
README.md
.vscode
.idea
coverage
tests
```

## Adaptation notes

| Framework | Base image | Build command | CMD |
|-----------|-----------|--------------|-----|
| Node/Express | `node:20-alpine` | `npm run build` | `node server.js` |
| Next.js | `node:20-alpine` | `npm run build` | `npm start` |
| Python/Flask | `python:3.12-slim` | `pip install -r requirements.txt` | `gunicorn app:app -b 0.0.0.0:3000` |
| Python/FastAPI | `python:3.12-slim` | `pip install -r requirements.txt` | `uvicorn app.main:app --host 0.0.0.0 --port 3000` |
| Go | `golang:1.22-alpine` / `alpine:3.19` | `go build -o /app/server .` | `./server` |
| Ruby/Rails | `ruby:3.3-slim` | `bundle install` | `rails server -b 0.0.0.0 -p 3000` |

For Python, replace the health check `wget` with `curl` or a Python script, and use `--no-cache-dir` on pip installs.
