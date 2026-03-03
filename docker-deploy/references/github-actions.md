# GitHub Actions Workflow Reference

## .github/workflows/deploy.yml

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

# Prevent concurrent deployments
concurrency:
  group: production
  cancel-in-progress: false

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  DEPLOY_PATH: /opt/app

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Build and push image
        run: |
          IMAGE_TAG="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}"
          docker build -t "${IMAGE_TAG}:${{ github.sha }}" -t "${IMAGE_TAG}:latest" .
          docker push "${IMAGE_TAG}:${{ github.sha }}"
          docker push "${IMAGE_TAG}:latest"

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push
    timeout-minutes: 10

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -p ${{ secrets.SERVER_PORT || 22 }} -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts 2>/dev/null

      - name: Copy deployment files
        run: |
          scp -i ~/.ssh/deploy_key -P ${{ secrets.SERVER_PORT || 22 }} \
            docker-compose.yml docker-compose.prod.yml \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }}:${{ env.DEPLOY_PATH }}/
          scp -i ~/.ssh/deploy_key -P ${{ secrets.SERVER_PORT || 22 }} -r \
            nginx/ \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }}:${{ env.DEPLOY_PATH }}/

      - name: Deploy on server
        run: |
          ssh -i ~/.ssh/deploy_key -p ${{ secrets.SERVER_PORT || 22 }} \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} << 'DEPLOY_SCRIPT'
          set -e
          cd ${{ env.DEPLOY_PATH }}

          # Authenticate with GHCR
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

          # Pull the new image
          export IMAGE_TAG="ghcr.io/${{ github.repository }}:${{ github.sha }}"
          docker pull "${IMAGE_TAG}"
          docker tag "${IMAGE_TAG}" app:latest

          # Deploy
          docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --force-recreate --remove-orphans

          # Wait for health checks
          echo "Waiting for health checks..."
          sleep 10
          docker compose ps --format json | grep -q '"Health":"healthy"' || \
            (echo "ERROR: Containers not healthy" && docker compose logs --tail=50 && exit 1)

          # Cleanup old images
          docker image prune -f
          DEPLOY_SCRIPT

      - name: Verify deployment
        run: |
          ssh -i ~/.ssh/deploy_key -p ${{ secrets.SERVER_PORT || 22 }} \
            ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} \
            'curl -sf http://localhost/health || (echo "Health check failed" && exit 1)'
          echo "Deployment verified successfully"

      - name: Cleanup SSH key
        if: always()
        run: rm -f ~/.ssh/deploy_key
```

## Required GitHub Secrets

Set these in the repo under Settings → Secrets and variables → Actions:

| Secret | Description | Example |
|--------|-------------|---------|
| `SERVER_HOST` | Server IP or hostname | `203.0.113.50` |
| `SERVER_USER` | SSH username | `deploy` |
| `SSH_PRIVATE_KEY` | Private key for SSH auth | Contents of `~/.ssh/id_ed25519` |
| `SERVER_PORT` | SSH port (optional, default 22) | `22` |

`GITHUB_TOKEN` is provided automatically by GitHub Actions and grants GHCR push/pull access.

## Server-side GHCR authentication

The deploy script logs into GHCR on the server using `GITHUB_TOKEN`. For private repos, the server needs pull access. The workflow handles this inline. No additional PATs are required.
