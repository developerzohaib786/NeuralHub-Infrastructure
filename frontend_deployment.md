# NeuralHub Frontend - Production Deployment Guide

Complete guide for deploying the NeuralHub Frontend (Next.js) on the SAME Ubuntu Server as backend using Docker and CI/CD with GitHub Actions.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Server Setup](#server-setup)
3. [Docker Installation](#docker-installation)
4. [Application Setup](#application-setup)
5. [Nginx Configuration](#nginx-configuration)
6. [CI/CD Pipeline Setup](#cicd-pipeline-setup)
7. [Deployment Process](#deployment-process)
8. [Monitoring & Maintenance](#monitoring--maintenance)
9. [Troubleshooting](#troubleshooting)
10. [Rollback Strategy](#rollback-strategy)

---

## Prerequisites

**IMPORTANT**: This guide assumes your backend is ALREADY deployed following `Backend_Deployment.md`. Your server should already have:

- ✅ `/opt/neuralhub/` directory structure
- ✅ `neuralhub` user created
- ✅ Repository cloned at `/opt/neuralhub/app`
- ✅ Docker installed
- ✅ Nginx installed
- ✅ SSL certificates configured (optional)

### Additional Requirements for Frontend
- Docker and Docker Compose already installed on server
- Domain name for frontend (can be same domain or subdomain)
- Clerk publishable key for authentication

---

## Server Setup

### 1. Install Docker (if not already installed)

SSH into your server:

```bash
ssh username@your-server-ip
```

Check if Docker is installed:

```bash
docker --version
```

If not installed, install Docker:

```bash
# Install Docker
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add neuralhub user to docker group
sudo usermod -aG docker neuralhub
```

### 2. Verify Directory Structure

Your directory structure should already exist from backend setup:

```bash
ls -la /opt/neuralhub/app/
# Should show: backend/, frontend/, .git/, etc.
```

If `/opt/neuralhub/logs/` doesn't exist, create it:

```bash
sudo mkdir -p /opt/neuralhub/logs
sudo chown -R neuralhub:neuralhub /opt/neuralhub
```

---

## Application Setup

**Note**: Your repository is already cloned at `/opt/neuralhub/app` from the backend setup. We'll just work with the frontend directory.

### 1. Test Docker Build

Switch to neuralhub user and test the build:

```bash
sudo su - neuralhub
cd ~/app/frontend

# Test build with sample environment variables
docker build \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="test-key" \
  --build-arg NEXT_PUBLIC_API_BASE="http://localhost:5003" \
  --build-arg NEXT_PUBLIC_API_URL="http://localhost:5003/api" \
  --build-arg NEXT_PUBLIC_WS_BASE="ws://localhost:5003" \
  --build-arg NEXT_PUBLIC_HUBROOM_URL="http://localhost:3000" \
  --build-arg BACKEND_URL="http://localhost:5003" \
  -t neuralhub-frontend:test .

# If build succeeds, remove test image
docker rmi neuralhub-frontend:test

# Exit back to your sudo user
exit
```

If the build fails, check the Docker logs and fix any issues before proceeding.

---

## Nginx Configuration

Your setup:
- **Frontend**: `https://neuralhub.us/` and `https://www.neuralhub.us/`
- **Backend**: `https://agent.neuralhub.us/`

### 1. Create Frontend Nginx Configuration

Create a new Nginx config file for the frontend:

```bash
sudo nano /etc/nginx/sites-available/neuralhub-frontend
```

Add this configuration:

```nginx
upstream neuralhub_frontend {
    server 127.0.0.1:3000;
}

# HTTP redirect to HTTPS
server {
    listen 80;
    server_name neuralhub.us www.neuralhub.us;
    return 301 https://$host$request_uri;
}

# HTTPS Frontend
server {
    listen 443 ssl http2;
    server_name neuralhub.us www.neuralhub.us;

    # SSL Configuration (will be configured by Certbot)
    # ssl_certificate /etc/letsencrypt/live/neuralhub.us/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/neuralhub.us/privkey.pem;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 100M;

    # Logging
    access_log /var/log/nginx/neuralhub-frontend-access.log;
    error_log /var/log/nginx/neuralhub-frontend-error.log;

    # Frontend - All routes
    location / {
        proxy_pass http://neuralhub_frontend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Next.js static files - cache aggressively
    location /_next/static/ {
        proxy_pass http://neuralhub_frontend;
        expires 365d;
        add_header Cache-Control "public, immutable";
    }

    # Public assets
    location /images/ {
        proxy_pass http://neuralhub_frontend;
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

### 2. Enable Frontend Site

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/neuralhub-frontend /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### 3. Setup SSL Certificate

Get SSL certificate for the frontend domain:

```bash
sudo certbot --nginx -d neuralhub.us -d www.neuralhub.us
```

Certbot will automatically update the Nginx config with SSL settings.

### 4. Verify Backend Configuration

Your backend should already be configured at `agent.neuralhub.us`. Verify it exists:

```bash
ls -la /etc/nginx/sites-enabled/ | grep neuralhub
```

If the backend Nginx config doesn't exist yet, create it:

```bash
sudo nano /etc/nginx/sites-available/neuralhub-backend
```

Add:

```nginx
upstream neuralhub_backend {
    server 127.0.0.1:5003;
}

# HTTP redirect to HTTPS
server {
    listen 80;
    server_name agent.neuralhub.us;
    return 301 https://$host$request_uri;
}

# HTTPS Backend
server {
    listen 443 ssl http2;
    server_name agent.neuralhub.us;

    # SSL Configuration (will be configured by Certbot)
    # ssl_certificate /etc/letsencrypt/live/agent.neuralhub.us/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/agent.neuralhub.us/privkey.pem;

    client_max_body_size 100M;

    # Logging
    access_log /var/log/nginx/neuralhub-backend-access.log;
    error_log /var/log/nginx/neuralhub-backend-error.log;

    location / {
        proxy_pass http://neuralhub_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Health check
    location /health {
        proxy_pass http://neuralhub_backend/health;
        access_log off;
    }
}
```

Enable backend:

```bash
sudo ln -s /etc/nginx/sites-available/neuralhub-backend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d agent.neuralhub.us
```

### 5. Final Verification

Test both domains:

```bash
# Test frontend
curl -I https://neuralhub.us/

# Test backend
curl -I https://agent.neuralhub.us/health
```

---

## CI/CD Pipeline Setup

### 1. GitHub Secrets Configuration

The GitHub Actions workflow is already updated in `.github/workflows/deploy-frontend.yml`. You need to add these secrets to your GitHub repository:

**Go to**: Repository → Settings → Secrets and variables → Actions

**Add these NEW secrets** (in addition to the backend secrets you already have):

| Secret Name | Value for neuralhub.us | Description |
|-------------|------------------------|-------------|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | `pk_live_xxxxx` | Your Clerk authentication key |
| `NEXT_PUBLIC_API_BASE` | `https://agent.neuralhub.us` | Backend API base URL |
| `NEXT_PUBLIC_API_URL` | `https://agent.neuralhub.us/api` | Full backend API URL |
| `NEXT_PUBLIC_WS_BASE` | `wss://agent.neuralhub.us` | WebSocket base URL |
| `NEXT_PUBLIC_HUBROOM_URL` | `https://neuralhub.us/hubroom` | HubRoom URL (if needed) |
| `BACKEND_URL` | `https://agent.neuralhub.us` | Backend URL for server-side requests |

**Note**: You should already have `SSH_PRIVATE_KEY`, `SERVER_IP`, and `SERVER_USER` from backend setup.

### 2. Configure Server Permissions

Your SSH user needs permission to run docker commands. Add this to sudoers:

```bash
sudo visudo
```

Update the line to include docker:

```
your-username ALL=(ALL) NOPASSWD: /usr/bin/supervisorctl, /usr/bin/git, /opt/neuralhub/.local/bin/uv, /usr/bin/docker
```

Or simply add your user to the docker group:

```bash
sudo usermod -aG docker your-username
```

### 3. Test Deployment

The workflow `.github/workflows/deploy-frontend.yml` is already configured. Test it:

```bash
# On your local machine
git add .
git commit -m "Test frontend deployment"
git push origin main
```

Check GitHub Actions tab to see the deployment running.

---

## Monitoring & Troubleshooting

### Check Container Status

```bash
# View running containers
sudo docker ps | grep neuralhub-frontend

# View container logs
sudo docker logs neuralhub-frontend --tail 100
sudo docker logs -f neuralhub-frontend  # Follow logs

# Restart container
sudo docker restart neuralhub-frontend

# Check resource usage
sudo docker stats neuralhub-frontend --no-stream
```

### Common Issues

**Container won't start:**
```bash
sudo docker logs neuralhub-frontend
sudo lsof -i :3000  # Check if port is in use
```

**Nginx 502 Bad Gateway:**
```bash
sudo docker ps | grep neuralhub-frontend  # Check if running
curl http://127.0.0.1:3000/  # Test directly
sudo tail -f /var/log/nginx/error.log
sudo systemctl reload nginx
```

**High memory usage:**
```bash
sudo docker stats neuralhub-frontend
sudo docker restart neuralhub-frontend
```

**Deployment failed:**
```bash
# Check GitHub Actions logs first
# Then on server:
cd /opt/neuralhub/app
sudo -u neuralhub git status
sudo -u neuralhub git pull origin main
```

### Manual Deployment (if CI/CD fails)

```bash
ssh username@your-server-ip

cd /opt/neuralhub/app
sudo -u neuralhub git pull origin main
cd frontend

# Build image
sudo docker build \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="your-key" \
  --build-arg NEXT_PUBLIC_API_BASE="https://yourdomain.com" \
  --build-arg NEXT_PUBLIC_API_URL="https://yourdomain.com/api" \
  --build-arg NEXT_PUBLIC_WS_BASE="wss://yourdomain.com" \
  --build-arg NEXT_PUBLIC_HUBROOM_URL="https://yourdomain.com/hubroom" \
  --build-arg BACKEND_URL="https://yourdomain.com" \
  -t neuralhub-frontend:latest .

# Stop old container
sudo docker stop neuralhub-frontend
sudo docker rm neuralhub-frontend

# Start new container
sudo docker run -d \
  --name neuralhub-frontend \
  --restart unless-stopped \
  -p 127.0.0.1:3000:3000 \
  neuralhub-frontend:latest

# Verify
sudo docker ps | grep neuralhub-frontend
curl http://localhost:3000/
```

---

## Summary

**What You Have Now:**
1. ✅ Frontend containerized with Docker
2. ✅ Nginx configured to serve frontend
3. ✅ GitHub Actions CI/CD for automatic deployment
4. ✅ Zero-downtime deployment with health checks
5. ✅ Integrated with existing backend setup

**Directory Structure:**
```
/opt/neuralhub/app/
├── backend/           # Backend (Supervisor-managed)
└── frontend/          # Frontend (Docker-managed)
```

**Services:**
- Backend API: `supervisorctl status neuralhub-api`
- Backend Worker: `supervisorctl status neuralhub-worker`
- Frontend: `docker ps | grep neuralhub-frontend`

**Deployment Flow:**
1. Push code to `main` branch (frontend/** changes)
2. GitHub Actions triggers
3. SSH into server
4. Pull latest code
5. Build Docker image
6. Replace container
7. Health check
8. Done ✅

**Need Help?**
- Check logs: `sudo docker logs neuralhub-frontend`
- Check Nginx: `sudo nginx -t && sudo systemctl status nginx`
- Check backend: `sudo supervisorctl status`

---

**Last Updated**: January 2026
