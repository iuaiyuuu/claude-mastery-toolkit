# Troubleshooting Reference

## Verification steps

Run these after every deployment to confirm everything works:

```bash
# 1. All containers running and healthy
docker compose ps
# Expected: all services show "Up" and "healthy"

# 2. App responds on internal port
docker compose exec app wget -qO- http://localhost:3000/health
# Expected: 200 OK or your health response

# 3. Nginx proxies correctly
curl -f http://localhost/health
# Expected: same response as step 2

# 4. Nginx returns proper headers
curl -I http://localhost
# Expected: X-Frame-Options, X-Content-Type-Options headers present

# 5. Check logs for errors
docker compose logs --tail=50 app
docker compose logs --tail=50 nginx
```

## Common failure checklist

### Container won't start

| Symptom | Cause | Fix |
|---------|-------|-----|
| `exited with code 1` | App crash on startup | Check logs: `docker compose logs app` |
| `port already in use` | Another process on port 80 | `sudo lsof -i :80` and stop it, or change Nginx port |
| `no such file or directory` | Missing .env or config | Verify `.env` exists in deploy path |
| `permission denied` | Wrong file ownership | `sudo chown -R deploy:deploy /opt/app` |

### Health check failing

| Symptom | Cause | Fix |
|---------|-------|-----|
| Container shows `unhealthy` | App not responding on /health | Ensure app has a `/health` endpoint returning 200 |
| Health check timeout | App takes too long to start | Increase `start_period` in docker-compose |
| `wget: connection refused` | App not listening on expected port | Verify `PORT` env var matches Dockerfile EXPOSE |

### Nginx issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `502 Bad Gateway` | App container not running or not healthy | Check `docker compose ps` — app must be healthy before Nginx starts |
| `502` after deploy | DNS resolution stale | `docker compose restart nginx` |
| `connection refused` on port 80 | Firewall blocking | `sudo ufw allow 80/tcp` |
| Static assets not loading | Wrong proxy path | Check `location` blocks in nginx config |

### GitHub Actions deployment fails

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Permission denied (publickey)` | SSH key mismatch | Verify `SSH_PRIVATE_KEY` secret matches server's authorized_keys |
| `Connection timed out` | Wrong host/port or firewall | Verify `SERVER_HOST`, `SERVER_PORT` secrets; check server firewall |
| `docker: command not found` | Deploy user not in docker group | `sudo usermod -aG docker deploy` then re-login |
| `docker pull` fails | Disk space full or GHCR auth failed | `docker system prune -af`; verify `GITHUB_TOKEN` has packages:read |

### Networking

| Symptom | Cause | Fix |
|---------|-------|-----|
| Containers can't reach each other | Not on same network | Verify both services use `app-network` in compose |
| App can't reach database | DB not healthy yet | Add `depends_on: db: condition: service_healthy` |
| External API calls fail from container | DNS resolution | Add `dns: [8.8.8.8, 1.1.1.1]` to service in compose |

## Emergency rollback

```bash
cd /opt/app

# List available local images
docker images --format "{{.Repository}}:{{.Tag}} {{.CreatedAt}}" | grep ghcr

# Rollback to previous SHA
docker pull ghcr.io/YOUR_ORG/YOUR_REPO:<previous-sha>
docker tag ghcr.io/YOUR_ORG/YOUR_REPO:<previous-sha> app:latest
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --force-recreate

# Or trigger rollback via git:
# git revert HEAD && git push origin main
```

## Log inspection

```bash
# Follow all logs
docker compose logs -f

# Specific service, last 100 lines
docker compose logs --tail=100 app

# With timestamps
docker compose logs -t app

# Nginx access log
docker compose exec nginx cat /var/log/nginx/access.log

# System-level Docker logs (if containers won't even start)
sudo journalctl -u docker --since "1 hour ago"
```
