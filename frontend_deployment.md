# NeuralHub Frontend - Production Deployment Guide

Complete guide for deploying the NeuralHub Next.js frontend on the same Ubuntu server as the backend, using Docker and CI/CD with GitHub Actions.

This guide assumes:
- The server is already set up per `backend_deployment.md` (Nginx, Supervisor, firewall, `neuralhub` user, `/opt/neuralhub/app` repo checkout already exist).
- **Docker is already installed** on the server.
- The frontend lives in a `frontend/` directory inside the same repository as the backend, with the `Dockerfile` shown below at its root.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [How This Differs From the Backend](#how-this-differs-from-the-backend)
3. [Application Setup](#application-setup)
4. [CI/CD Pipeline Setup](#cicd-pipeline-setup)
5. [Nginx & Domain Configuration](#nginx--domain-configuration)
6. [Deployment Process](#deployment-process)
7. [Monitoring & Maintenance](#monitoring--maintenance)
8. [Troubleshooting](#troubleshooting)
9. [Rollback Strategy](#rollback-strategy)

---

## Prerequisites

### Local Requirements
- Git with SSH key configured for GitHub
- Access to the same Ubuntu server used for the backend

### Server Requirements
- Docker installed and running (`sudo docker --version` to confirm)
- Same `/opt/neuralhub/app` repository checkout used by the backend (frontend code lives at `/opt/neuralhub/app/frontend`)
- Nginx already installed (from backend setup)
- Ports 80/443 open in the firewall (already done in backend setup)

### Build-Time Environment Variables

Your `Dockerfile` bakes these `NEXT_PUBLIC_*` values into the build at build time (Next.js requirement — they can't be injected at runtime like normal env vars):

| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk auth public key |
| `NEXT_PUBLIC_API_BASE` | Base URL the browser calls for the API |
| `NEXT_PUBLIC_API_URL` | Full API URL used client-side |
| `NEXT_PUBLIC_WS_BASE` | WebSocket base URL |
| `NEXT_PUBLIC_HUBROOM_URL` | Hubroom service URL |
| `BACKEND_URL` | Server-side backend URL (used during SSR/build) |

Because these are baked in at build time, **any change to these values requires rebuilding the image**, not just restarting the container.

---

## How This Differs From the Backend

The backend uses Supervisor to run a Python process directly on the host. The frontend instead runs **inside a Docker container** built from the multi-stage `Dockerfile` (Node 22 Alpine, Next.js standalone output, non-root `nextjs` user, listening on port `3000`).

Consequences:
- No `venv`, no `uv`, no Supervisor config for the frontend — Docker handles the process lifecycle via `--restart unless-stopped`.
- The container binds only to `127.0.0.1:3000` on the host; Nginx is the only thing exposed publicly, exactly like the backend on port `5003`.
- Deployments rebuild the image and swap the container, rather than doing a `git pull` + in-place restart.

---

## Application Setup

### 1. Confirm the Repository Layout

The frontend code should already be present from the initial clone described in `backend_deployment.md`:

```bash
sudo su - neuralhub
cd /opt/neuralhub/app/frontend
ls
# Should show: Dockerfile, package.json, package-lock.json, app/ (or src/), public/, etc.
exit
```

If the frontend hasn't been pulled yet, a normal `git pull origin main` from `/opt/neuralhub/app` will bring it in alongside the backend.

### 2. Verify Docker Access

Since deployment runs `sudo docker ...` over SSH, confirm the deploying user (your `SERVER_USER`, **not** `neuralhub`) can run Docker commands:

```bash
docker --version
sudo docker run hello-world
```

### 3. One-Time Manual Build (Sanity Check)

Before wiring up CI/CD, do a manual build to confirm the `Dockerfile` and env vars are correct:

```bash
cd /opt/neuralhub/app/frontend

sudo docker build \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="pk_live_xxx" \
  --build-arg NEXT_PUBLIC_API_BASE="https://api.your-domain.com" \
  --build-arg NEXT_PUBLIC_API_URL="https://api.your-domain.com" \
  --build-arg NEXT_PUBLIC_WS_BASE="wss://api.your-domain.com" \
  --build-arg NEXT_PUBLIC_HUBROOM_URL="https://hubroom.your-domain.com" \
  --build-arg BACKEND_URL="http://127.0.0.1:5003" \
  -t neuralhub-frontend:latest .

sudo docker run -d --name neuralhub-frontend --restart unless-stopped \
  -p 127.0.0.1:3000:3000 neuralhub-frontend:latest

curl -f http://localhost:3000/
```

If `curl` returns HTML, the container is healthy. Stop here and clean up before moving to CI/CD if you were just testing:

```bash
sudo docker stop neuralhub-frontend && sudo docker rm neuralhub-frontend
```

---

## CI/CD Pipeline Setup

### 1. Add the Workflow File

Place the workflow at `.github/workflows/deploy-frontend.yml` in your repository (provided alongside this guide). It:

- Triggers on pushes to `main` that touch `frontend/**`, or manually via `workflow_dispatch`
- SSHes into the server, pulls the latest code
- Builds a fresh Docker image with the `NEXT_PUBLIC_*` build args from GitHub Secrets
- Tags the previous image as `neuralhub-frontend:previous` for rollback
- Stops/removes the old container and starts a new one bound to `127.0.0.1:3000`
- Runs a health check against `http://localhost:3000/`
- Prunes dangling images to save disk space

### 2. Setup GitHub Secrets

Go to your repository → **Settings → Secrets and variables → Actions**, and add:

**Shared with backend (already set if you followed `backend_deployment.md`):**
- `SSH_PRIVATE_KEY`
- `SERVER_IP`
- `SERVER_USER`

**Frontend-specific build args:**
- `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
- `NEXT_PUBLIC_API_BASE`
- `NEXT_PUBLIC_API_URL`
- `NEXT_PUBLIC_WS_BASE`
- `NEXT_PUBLIC_HUBROOM_URL`
- `BACKEND_URL`

### 3. Confirm Sudo Access for Docker

Your `SERVER_USER` needs passwordless sudo for Docker (and `git`, if not already granted for backend deploys):

```bash
sudo visudo
```

Add (or extend the existing backend line):

```
your-username ALL=(ALL) NOPASSWD: /usr/bin/supervisorctl, /usr/bin/git, /usr/bin/docker
```

### 4. Test the Pipeline

```bash
git add .github/workflows/deploy-frontend.yml frontend_deployment.md
git commit -m "Add frontend CI/CD deployment pipeline"
git push origin main
```

Watch it run under your repository's **Actions** tab.

---

## Nginx & Domain Configuration

The frontend container listens only on `127.0.0.1:3000`. Nginx is what the public domain actually points to, exactly as with the backend on port `5003`.

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
    server_name your-domain.com www.your-domain.com;  # Replace with your actual domain

    client_max_body_size 20M;

    location / {
        proxy_pass http://neuralhub_frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required for Next.js hot-reload/websocket-based features
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Next.js static assets — safe to cache aggressively
    location /_next/static/ {
        proxy_pass http://neuralhub_frontend;
        proxy_cache_valid 200 60m;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

> **If your API runs on a separate subdomain** (e.g. `api.your-domain.com` pointing at the backend's port `5003`), keep that as its own server block — reuse the one already defined in `backend_deployment.md`. This file only needs to handle the domain(s) that should serve the frontend.

### 2. Enable the Site

```bash
sudo ln -s /etc/nginx/sites-available/neuralhub-frontend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 3. Point Your Domain's DNS

At your DNS provider, create an **A record** (and optionally an **AAAA** for IPv6):

| Type | Name | Value |
|---|---|---|
| A | `@` (or `your-domain.com`) | Your server's public IP |
| A | `www` | Your server's public IP |

DNS propagation can take a few minutes to a few hours.

### 4. Setup SSL with Let's Encrypt

If Certbot isn't already installed from the backend setup:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Issue and auto-configure the certificate:

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

Certbot rewrites the site config to redirect HTTP → HTTPS and sets up auto-renewal (`systemctl status certbot.timer` to confirm).

### 5. Verify

```bash
curl -I https://your-domain.com
```

You should see a `200 OK` (or a Next.js redirect) served over HTTPS.

---

## Deployment Process

### Automatic Deployment

Every push to `main` touching `frontend/**` rebuilds the image and redeploys the container automatically.

### Manual Deployment

1. Repository → **Actions**
2. Select **"Deploy Frontend to Ubuntu Server"**
3. **Run workflow** → choose branch → **Run workflow**

### Monitoring a Deployment

```bash
# Container logs
sudo docker logs -f neuralhub-frontend

# Container status
sudo docker ps --filter "name=neuralhub-frontend"
```

---

## Monitoring & Maintenance

### Container Management

```bash
# Status
sudo docker ps -a --filter "name=neuralhub-frontend"

# Restart
sudo docker restart neuralhub-frontend

# Stop / Start
sudo docker stop neuralhub-frontend
sudo docker start neuralhub-frontend

# Live logs
sudo docker logs -f --tail 200 neuralhub-frontend

# Shell into the running container (debugging)
sudo docker exec -it neuralhub-frontend sh
```

### Health Checks

```bash
curl http://localhost:3000/
sudo docker inspect -f '{{.State.Health.Status}}' neuralhub-frontend 2>/dev/null || echo "no healthcheck defined"
```

### Disk Usage — Docker Images

Old/dangling images accumulate over time. The CI workflow prunes dangling images automatically, but you can also check manually:

```bash
sudo docker images
sudo docker system df
sudo docker image prune -af --filter "until=168h"   # remove unused images older than 7 days
```

### Log Rotation

Docker's default `json-file` logging driver can grow unbounded. Cap it globally:

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
sudo docker restart neuralhub-frontend
```

---

## Troubleshooting

### Container Won't Start

```bash
sudo docker logs neuralhub-frontend
```

Common causes: a missing/incorrect `NEXT_PUBLIC_*` build arg baked into a bad build, or port `3000` already in use.

```bash
sudo lsof -i :3000
```

### Build Fails on the Server

SSH in and reproduce the build directly to see the full error:

```bash
cd /opt/neuralhub/app/frontend
sudo docker build -t neuralhub-frontend:debug .
```

### 502 Bad Gateway from Nginx

Usually means the container isn't running or isn't listening on `3000`:

```bash
sudo docker ps --filter "name=neuralhub-frontend"
curl http://127.0.0.1:3000/
sudo tail -f /var/log/nginx/error.log
```

### Old Content Still Showing After Deploy

Check the browser isn't serving a cached version, and confirm the new image is actually running:

```bash
sudo docker inspect --format='{{.Created}}' neuralhub-frontend
sudo docker images neuralhub-frontend
```

### SSL Certificate Issues

```bash
sudo certbot certificates
sudo certbot renew --dry-run
```

---

## Rollback Strategy

The deploy workflow tags the previous image as `neuralhub-frontend:previous` before building the new one, so rollback doesn't require a rebuild.

### Manual Rollback (Fast — Reuse Previous Image)

```bash
ssh username@your-server-ip

sudo docker stop neuralhub-frontend
sudo docker rm neuralhub-frontend

sudo docker run -d \
  --name neuralhub-frontend \
  --restart unless-stopped \
  -p 127.0.0.1:3000:3000 \
  neuralhub-frontend:previous

curl -f http://localhost:3000/
```

### Full Rollback (Rebuild From an Older Commit)

```bash
cd /opt/neuralhub/app
sudo -u neuralhub git log --oneline -5
sudo -u neuralhub git reset --hard COMMIT_HASH

cd frontend
sudo docker build \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="..." \
  --build-arg NEXT_PUBLIC_API_BASE="..." \
  --build-arg NEXT_PUBLIC_API_URL="..." \
  --build-arg NEXT_PUBLIC_WS_BASE="..." \
  --build-arg NEXT_PUBLIC_HUBROOM_URL="..." \
  --build-arg BACKEND_URL="..." \
  -t neuralhub-frontend:latest .

sudo docker stop neuralhub-frontend && sudo docker rm neuralhub-frontend
sudo docker run -d --name neuralhub-frontend --restart unless-stopped \
  -p 127.0.0.1:3000:3000 neuralhub-frontend:latest
```

> **Note:** Because `main.reset --hard` rewrites the working tree shared with the backend checkout, coordinate frontend rollbacks with whoever manages backend deploys — resetting the repo affects both.

---

## Summary Checklist

### Initial Setup
- [ ] Docker confirmed installed and working on the server
- [ ] Frontend code present at `/opt/neuralhub/app/frontend`
- [ ] Manual `docker build` + `docker run` sanity check passed
- [ ] Nginx site config created and enabled
- [ ] DNS A record(s) pointed at the server
- [ ] SSL certificate issued via Certbot

### CI/CD Setup
- [ ] `.github/workflows/deploy-frontend.yml` added to the repo
- [ ] All `NEXT_PUBLIC_*` and `BACKEND_URL` secrets set in GitHub
- [ ] Passwordless sudo for `docker` confirmed for the deploy user
- [ ] Pipeline tested end-to-end from a push to `main`

### Production Readiness
- [ ] Health check passing at `https://your-domain.com`
- [ ] Docker log rotation configured
- [ ] Rollback tested at least once
- [ ] Old/dangling images pruned automatically

---

**Last Updated**: August 2026
**Version**: 1.0
**Maintained by**: NeuralHub Team
