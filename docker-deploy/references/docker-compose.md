# Docker Compose Reference

## docker-compose.yml

```yaml
services:
  app:
    image: app:latest
    container_name: app
    env_file:
      - .env
    expose:
      - "3000"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    restart: unless-stopped
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      app:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## docker-compose.prod.yml

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"

  nginx:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## .env.example

```bash
# Copy to .env and fill in real values. Never commit .env to git.
NODE_ENV=production
PORT=3000
APP_SECRET=change-me-to-a-real-secret
```

## Database addendum

Only include if the user requests a database. Add to docker-compose.yml:

```yaml
  db:
    image: postgres:16-alpine
    container_name: db
    env_file:
      - .env
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - app-network

volumes:
  db-data:
```

Add `depends_on: db: condition: service_healthy` to the app service. Add DB vars to .env.example:

```bash
DB_HOST=db
DB_PORT=5432
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=change-me
```
