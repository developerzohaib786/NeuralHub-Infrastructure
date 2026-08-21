# NeuralHub Frontend - Production Deployment Guide

Complete guide for deploying the NeuralHub Next.js frontend as a Docker container on the same Ubuntu server (`vmi3400404`) that runs the backend, with CI/CD via GitHub Actions and Nginx handling the domain.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Server Setup for Docker](#server-setup-for-docker)
4. [Environment File on the Server](#environment-file-on-the-server)
5. [CI/CD Pipeline Setup](#cicd-pipeline-setup)
6. [Nginx + Domain Setup](#nginx--domain-setup)
7. [Deployment Process](#deployment-process)
8. [Monitoring & Maintenance](#monitoring--maintenance)
9. [Troubleshooting](#troubleshooting)
10. [Rollback Strategy](#rollback-strategy)

---

## Overview

Unlike the backend (run directly via `uv` + Supervisor), the frontend ships as a **Docker image** built from the multi-stage `Dockerfile` in `frontend/`. The pipeline:

1. GitHub Actions builds the image (with `NEXT_PUBLIC_*` build args baked in at build time, since Next.js inlines these into the client bundle).
2. The image is pushed to **GitHub Container Registry (GHCR)** — free for repos already on GitHub, no extra account needed.
3. GitHub Actions SSHes into the server, pulls the new image, and swaps the running container.
4. Nginx (already installed for the backend) reverse-proxies the domain to the container on `127.0.0.1:3000`.

This keeps the server itself free of Node/npm — it only needs Docker.

---

## Prerequisites

- Server already set up per the backend deployment guide (Ubuntu, Nginx, Certbot, firewall).
- Docker installed on the server (steps below).
- A domain or subdomain for the frontend (e.g. `app.neuralhub.us` or the root `neuralhub.us`), pointed at the server's IP via an A record. This is separate from the backend's `agent.neuralhub.us`.
- GitHub repository access with permission to add secrets and enable GHCR package publishing.

---

## Server Setup for Docker

### 1. Install Docker Engine

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify:

```bash
sudo docker --version
```

### 2. Allow Your SSH User to Run Docker (optional, avoids `sudo` everywhere)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

The CI workflow below still uses `sudo docker ...` for safety on a fresh shell session — drop the `sudo` once your user is in the `docker` group and re-logged in.

### 3. Create the Frontend Directory

```bash
sudo mkdir -p /opt/neuralhub/frontend
sudo chown -R $USER:$USER /opt/neuralhub/frontend
```

This only holds the `.env.production` file — the actual app code lives inside the Docker image, not on disk.

---

## Environment File on the Server

Next.js build-time variables (`NEXT_PUBLIC_*`) are baked into the image at **build** time via `--build-arg` in CI. Anything the container needs at **runtime** only (e.g. server-only secrets not prefixed `NEXT_PUBLIC_`) goes in an env file on the server, referenced by `--env-file` when the container starts:

```bash
nano /opt/neuralhub/frontend/.env.production
```

```env
BACKEND_URL=https://agent.neuralhub.us
# add any server-only (non NEXT_PUBLIC_) runtime variables here
```

Lock it down:

```bash
chmod 600 /opt/neuralhub/frontend/.env.production
```

---

## CI/CD Pipeline Setup

### 1. Add the Workflow File

Save the workflow as `.github/workflows/deploy-frontend.yml` in your repo (content below — triggers only on changes under `frontend/`, mirroring the backend workflow's `paths` filter):

```yaml
name: Deploy Frontend to Ubuntu Server

on:
  push:
    branches:
      - main
    paths:
      - 'frontend/**'
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}-frontend
  CONTAINER_NAME: neuralhub-frontend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.image_tag }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set Image Tag
        id: meta
        run: echo "image_tag=${{ github.sha }}" >> "$GITHUB_OUTPUT"

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Image
        uses: docker/build-push-action@v6
        with:
          context: ./frontend
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.image_tag }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          build-args: |
            NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=${{ secrets.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY }}
            NEXT_PUBLIC_API_BASE=${{ secrets.NEXT_PUBLIC_API_BASE }}
            NEXT_PUBLIC_API_URL=${{ secrets.NEXT_PUBLIC_API_URL }}
            NEXT_PUBLIC_WS_BASE=${{ secrets.NEXT_PUBLIC_WS_BASE }}
            NEXT_PUBLIC_HUBROOM_URL=${{ secrets.NEXT_PUBLIC_HUBROOM_URL }}
            BACKEND_URL=${{ secrets.BACKEND_URL }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Setup SSH Key
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Add Server to Known Hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts

      - name: Deploy Frontend Container
        env:
          SERVER_USER: ${{ secrets.SERVER_USER }}
          SERVER_IP: ${{ secrets.SERVER_IP }}
          IMAGE: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-push.outputs.image_tag }}
          GHCR_USER: ${{ github.actor }}
          GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          ssh $SERVER_USER@$SERVER_IP << ENDSSH
            set -e

            echo "Logging in to GHCR..."
            echo "$GHCR_TOKEN" | sudo docker login ghcr.io -u "$GHCR_USER" --password-stdin

            echo "Pulling new image..."
            sudo docker pull $IMAGE

            echo "Stopping old container (if running)..."
            sudo docker stop ${{ env.CONTAINER_NAME }} || true
            sudo docker rm ${{ env.CONTAINER_NAME }} || true

            echo "Starting new container..."
            sudo docker run -d \
              --name ${{ env.CONTAINER_NAME }} \
              --restart unless-stopped \
              -p 127.0.0.1:3000:3000 \
              --env-file /opt/neuralhub/frontend/.env.production \
              $IMAGE

            echo "Waiting for container to come up..."
            sleep 5

            echo "Pruning old images..."
            sudo docker image prune -f

            echo "Health check..."
            curl -f http://127.0.0.1:3000 || exit 1

            echo "Frontend deployment completed successfully!"
          ENDSSH

      - name: Notify Deployment Success
        if: success()
        run: echo "Frontend deployed successfully to production!"

      - name: Notify Deployment Failure
        if: failure()
        run: |
          echo "Frontend deployment failed!"
          exit 1
```

**Why GHCR instead of building directly on the server (unlike the backend's approach)?** A Next.js Docker build (`npm ci` + `npm run build`) is CPU/RAM-heavy. Running it on a small production VPS alongside the live API can cause memory pressure — this server has already hit an out-of-memory issue once from an oversized Supervisor worker count. Building on GitHub's runners and only *pulling* the finished image keeps the server load-free during deploys.

### 2. GitHub Secrets

Repo → Settings → Secrets and variables → Actions → New repository secret:

| Secret | Purpose |
|---|---|
| `SSH_PRIVATE_KEY` | Private key with access to the server (same one used for the backend workflow, or a dedicated one) |
| `SERVER_IP` | Server IP or domain |
| `SERVER_USER` | SSH user (not `neuralhub` — same convention as backend) |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk publishable key |
| `NEXT_PUBLIC_API_BASE` | Public API base URL |
| `NEXT_PUBLIC_API_URL` | Public API URL |
| `NEXT_PUBLIC_WS_BASE` | WebSocket base URL |
| `NEXT_PUBLIC_HUBROOM_URL` | Hubroom URL |
| `BACKEND_URL` | Internal backend URL, e.g. `https://agent.neuralhub.us` |

`GITHUB_TOKEN` for GHCR login is provided automatically by Actions — no need to create it.

### 3. Allow GHCR Pulls from the Server

The first `docker login ghcr.io` inside the deploy step handles authentication per-run using the ephemeral `GITHUB_TOKEN`, so no persistent credentials sit on the server. If your repo's packages are private, make sure the token's associated GitHub App/PAT has `read:packages` — the default `GITHUB_TOKEN` already does for same-repo packages.

### 4. Sudo Without Password for Docker (optional)

If you kept `sudo docker` in the workflow rather than adding the SSH user to the `docker` group:

```bash
sudo visudo
```

```
your-username ALL=(ALL) NOPASSWD: /usr/bin/docker
```

### 5. Test the Pipeline

```bash
git add .github/workflows/deploy-frontend.yml
git commit -m "Add frontend Docker CI/CD pipeline"
git push origin main
```

Watch it run under GitHub → Actions.

---

## Nginx + Domain Setup

The frontend container listens on `127.0.0.1:3000` (not exposed publicly). Nginx handles the public domain, TLS, and proxying — the same pattern already used for `agent.neuralhub.us`.

### 1. Create the Nginx Site Config

```bash
sudo nano /etc/nginx/sites-available/neuralhub-frontend
```

```nginx
upstream neuralhub_frontend {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    server_name your-frontend-domain.com www.your-frontend-domain.com;  # e.g. app.neuralhub.us

    client_max_body_size 20M;

    location / {
        proxy_pass http://neuralhub_frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Next.js dev overlay / HMR & any WebSocket usage
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Serve static assets with long cache headers (Next.js standalone build)
    location /_next/static/ {
        proxy_pass http://neuralhub_frontend;
        proxy_cache_valid 200 60m;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

### 2. Enable the Site

```bash
sudo ln -s /etc/nginx/sites-available/neuralhub-frontend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 3. Point DNS at the Server

At your domain registrar / DNS provider, add an **A record** for the frontend domain (e.g. `app.neuralhub.us` or `@` for the root) pointing to the server's public IP — the same IP already used for `agent.neuralhub.us`.

### 4. Get an SSL Certificate

```bash
sudo certbot --nginx -d your-frontend-domain.com -d www.your-frontend-domain.com
```

Certbot edits the site config to add the HTTPS `server` block and 80→443 redirect, and sets up auto-renewal (shared with the backend's existing Certbot install, since it's the same server).

### 5. Verify

```bash
curl -I https://your-frontend-domain.com
```

You should get a `200` from the Next.js app.

---

## Deployment Process

### Automatic

Any push to `main` touching `frontend/**` triggers `build-and-push` then `deploy` automatically.

### Manual

GitHub → Actions → "Deploy Frontend to Ubuntu Server" → Run workflow.

### Watching a Deploy

```bash
sudo docker logs -f neuralhub-frontend
```

---

## Monitoring & Maintenance

### Container Management

```bash
# Status
sudo docker ps -a --filter name=neuralhub-frontend

# Logs
sudo docker logs --tail 100 -f neuralhub-frontend

# Restart
sudo docker restart neuralhub-frontend

# Stop / start
sudo docker stop neuralhub-frontend
sudo docker start neuralhub-frontend

# Resource usage
sudo docker stats neuralhub-frontend
```

### Image Cleanup

Old images accumulate on the server with every deploy. The workflow already runs `docker image prune -f` after each deploy; to clean up manually:

```bash
sudo docker image prune -af --filter "until=168h"   # anything older than 7 days
```

### Health Check

```bash
curl http://127.0.0.1:3000
```

---

## Troubleshooting

### Container Won't Start

```bash
sudo docker logs neuralhub-frontend
```

Common causes: missing/incorrect `NEXT_PUBLIC_*` build args baked into a bad image (fix the secret and re-run the workflow — you cannot fix this at runtime, since these are compiled into the client bundle), or a bad `.env.production` path.

### 502 Bad Gateway from Nginx

Means Nginx can't reach the container:

```bash
sudo docker ps | grep neuralhub-frontend   # is it running?
curl http://127.0.0.1:3000                 # does it respond locally?
sudo tail -f /var/log/nginx/error.log
```

### Port Already in Use

```bash
sudo lsof -i :3000
```

If something else already owns `3000`, either stop it or change the host-side port mapping (`-p 127.0.0.1:3001:3000`) and update the Nginx `upstream` block to match.

### GHCR Pull / Auth Failures on the Server

```bash
sudo docker login ghcr.io -u <github-username>
```

Confirm the package's visibility (repo → Packages) — private packages need the deploy step's token to have package-read access, which the default `GITHUB_TOKEN` has for same-repo packages.

### Old Version Still Showing After Deploy

Usually a browser/CDN cache issue for static assets, or Nginx `proxy_cache` (not enabled by default in the config above). Hard-refresh, or confirm the running container's image digest:

```bash
sudo docker inspect --format='{{.Image}}' neuralhub-frontend
```

---

## Rollback Strategy

### Manual Rollback

Every image is tagged with the commit SHA, so rolling back is just running the previous tag:

```bash
# Find previous good SHA from GitHub Actions history or:
sudo docker images | grep neuralhub-agents-frontend

sudo docker stop neuralhub-frontend
sudo docker rm neuralhub-frontend

sudo docker run -d \
  --name neuralhub-frontend \
  --restart unless-stopped \
  -p 127.0.0.1:3000:3000 \
  --env-file /opt/neuralhub/frontend/.env.production \
  ghcr.io/<owner>/<repo>-frontend:<previous-commit-sha>
```

### Rollback Workflow

Create `.github/workflows/rollback-frontend.yml`, mirroring the backend's rollback workflow but swapping the git-reset step for a `docker run` against a prior image tag:

```yaml
name: Rollback Frontend

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Commit SHA of the image to roll back to'
        required: true
        type: string

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Setup SSH Key
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Add Server to Known Hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts

      - name: Rollback Frontend Container
        env:
          SERVER_USER: ${{ secrets.SERVER_USER }}
          SERVER_IP: ${{ secrets.SERVER_IP }}
          IMAGE: ghcr.io/${{ github.repository }}-frontend:${{ github.event.inputs.image_tag }}
          GHCR_USER: ${{ github.actor }}
          GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          ssh $SERVER_USER@$SERVER_IP << ENDSSH
            set -e

            echo "$GHCR_TOKEN" | sudo docker login ghcr.io -u "$GHCR_USER" --password-stdin
            sudo docker pull $IMAGE

            sudo docker stop neuralhub-frontend || true
            sudo docker rm neuralhub-frontend || true

            sudo docker run -d \
              --name neuralhub-frontend \
              --restart unless-stopped \
              -p 127.0.0.1:3000:3000 \
              --env-file /opt/neuralhub/frontend/.env.production \
              $IMAGE

            sleep 5
            curl -f http://127.0.0.1:3000 || exit 1

            echo "Rollback completed successfully!"
          ENDSSH
```

---

## Summary Checklist

### Server Setup
- [ ] Docker Engine installed
- [ ] `/opt/neuralhub/frontend/.env.production` created and locked to `600`
- [ ] Nginx site config created and enabled
- [ ] DNS A record pointed at the server
- [ ] SSL certificate obtained via Certbot

### CI/CD Setup
- [ ] `.github/workflows/deploy-frontend.yml` added
- [ ] GitHub secrets configured (SSH + `NEXT_PUBLIC_*` + `BACKEND_URL`)
- [ ] First pipeline run completed successfully
- [ ] Rollback workflow added

### Production Readiness
- [ ] `https://your-frontend-domain.com` loads over HTTPS
- [ ] Health check passes (`curl http://127.0.0.1:3000` on server)
- [ ] Old images pruned automatically after each deploy
- [ ] Rollback tested at least once

---

**Maintained by**: NeuralHub Team
