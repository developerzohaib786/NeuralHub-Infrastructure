# NeuralHub Backend - Production Deployment Guide

Complete guide for deploying the NeuralHub Backend on Ubuntu Server with CI/CD using GitHub Actions.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Server Setup](#server-setup)
3. [Application Setup](#application-setup)
4. [CI/CD Pipeline Setup](#cicd-pipeline-setup)
5. [Deployment Process](#deployment-process)
6. [Monitoring & Maintenance](#monitoring--maintenance)
7. [Troubleshooting](#troubleshooting)
8. [Rollback Strategy](#rollback-strategy)

---

## Prerequisites

### Local Requirements
- Git with SSH key configured for GitHub
- Access to Ubuntu server (SSH)
- GitHub account with repository access

### Server Requirements
- Ubuntu Server 20.04 LTS or newer (22.04 recommended)
- Minimum 2GB RAM, 2 CPU cores
- 20GB+ storage
- Root or sudo access
- Public IP address or domain name

### External Services
- PostgreSQL database (Neon, AWS RDS, or self-hosted)
- Redis instance (Upstash, AWS ElastiCache, or self-hosted)
- Qdrant vector database (Qdrant Cloud)
- Cloudinary account
- API Keys: Cohere, OpenAI, Gemini, Groq

---

## Server Setup

### 1. Initial Server Configuration

SSH into your Ubuntu server:

```bash
ssh username@your-server-ip
```

Update system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Software

#### Install Python 3.11+

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3.11-dev python3-pip
```

Verify installation:

```bash
python3.11 --version
```

#### Install UV (Python Package Manager)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Note**: UV will be installed to `/opt/neuralhub/.local/bin/uv` (not `.cargo/bin`).

Verify installation:

```bash
/opt/neuralhub/.local/bin/uv --version
```

#### Install PostgreSQL Client (for database migrations)

```bash
sudo apt install -y postgresql-client
```

#### Install Nginx (Reverse Proxy)

```bash
sudo apt install -y nginx
```

#### Install Supervisor (Process Manager)

```bash
sudo apt install -y supervisor
```

### 3. Create Application User

Create a dedicated user for running the application:

```bash
# Create system user with bash shell access
sudo useradd -r -m -d /opt/neuralhub -s /bin/bash neuralhub
sudo usermod -aG www-data neuralhub
```

**Note**: We use `/bin/bash` instead of the default `/usr/sbin/nologin` to allow shell access for setup tasks.

### 4. Setup Application Directory

```bash
sudo mkdir -p /opt/neuralhub/app
sudo mkdir -p /opt/neuralhub/logs
sudo chown -R neuralhub:neuralhub /opt/neuralhub
```

### 5. Configure Firewall

```bash
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

---

## Application Setup

### 1. Clone Repository

Switch to neuralhub user:

```bash
sudo su - neuralhub
cd /opt/neuralhub
```

Generate SSH key for GitHub (if not already done):

```bash
ssh-keygen -t ed25519 -C "neuralhub-server"
cat ~/.ssh/id_ed25519.pub
```

Add this public key to your GitHub repository's Deploy Keys (Settings → Deploy Keys).

Clone the repository:

```bash
git clone git@github.com:yourusername/rag-chat-app.git app
cd app/backend
```

### 2. Setup Python Environment

Install dependencies using UV (automatically manages Python version and virtual environment):

```bash
# This will sync dependencies and create .venv automatically
uv sync --frozen

# UV automatically manages the virtual environment, no need to manually activate it
# When you use 'uv run', it automatically uses the .venv
```

**Understanding UV Workflow**:
- `uv sync`: Installs all dependencies from `uv.lock` and creates `.venv`
- `uv run <command>`: Runs command using the virtual environment automatically
- No need to manually activate virtual environment when using `uv run`

### 3. Configure Environment Variables

Create production `.env` file:

```bash
nano .env
```

Add your production configuration:

```env
# API Keys
OPENAI_API_KEY=sk-your-openai-key
GEMINI_API_KEY=your-gemini-key
COHERE_API_KEY=your-cohere-key
GROQ_API_KEY=your-groq-key

# Vector Database
QDRANT_API_KEY=your-qdrant-key
QDRANT_URL=https://your-qdrant-instance.qdrant.io

# Cloudinary
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret
CLOUDINARY_URL=cloudinary://your-full-url

# Database
DATABASE_URL=postgresql://user:password@host:port/database?sslmode=require

# Redis
REDIS_URL=rediss://default:password@host:port

# Server
PORT=5003
```

**Security Note**: Set proper file permissions:

```bash
chmod 600 .env
```

### 4. Run Database Migrations

Generate Prisma client and push schema:

```bash
# Generate Prisma client
uv run prisma generate

# Push database schema (creates tables)
uv run prisma db push

# Alternative: Use migrations for production
# uv run prisma migrate deploy
```

**Note**: `prisma db push` is simpler for initial setup, but `prisma migrate deploy` is recommended for production to track schema changes.

### 5. Test Application

Test that the application starts:

```bash
# Run with uv (automatically uses the correct Python and dependencies)
uv run uvicorn src.main:app --host 0.0.0.0 --port 5003
```

You should see output like:
```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:5003
```

Press `Ctrl+C` to stop.

**Test the worker** (open a new terminal):

```bash
# Switch to neuralhub user and navigate to backend
sudo su - neuralhub
cd ~/app/backend

# Run the worker
uv run python -m src.worker_runner
```

You should see the worker start successfully. Press `Ctrl+C` to stop.

If both services start successfully, proceed to process management.

---

## Process Management with Supervisor

### 1. Create Supervisor Configuration for API Server

Exit neuralhub user back to your sudo user:

```bash
exit
```

Create supervisor config:

```bash
sudo nano /etc/supervisor/conf.d/neuralhub-api.conf
```

Add configuration:

```ini
[program:neuralhub-api]
command=/opt/neuralhub/app/backend/.venv/bin/uvicorn src.main:app --host 0.0.0.0 --port 5003 --workers 4
directory=/opt/neuralhub/app/backend
user=neuralhub
autostart=true
autorestart=true
startretries=3
stderr_logfile=/opt/neuralhub/logs/api-error.log
stdout_logfile=/opt/neuralhub/logs/api-access.log
environment=PATH="/opt/neuralhub/app/backend/.venv/bin:/opt/neuralhub/.local/bin"
```

**Note**: We include both the virtual environment bin and local bin (for uv) in the PATH.

### 2. Create Supervisor Configuration for Worker

```bash
sudo nano /etc/supervisor/conf.d/neuralhub-worker.conf
```

Add configuration:

```ini
[program:neuralhub-worker]
command=/opt/neuralhub/app/backend/.venv/bin/python -m src.worker_runner
directory=/opt/neuralhub/app/backend
user=neuralhub
autostart=true
autorestart=true
startretries=3
stderr_logfile=/opt/neuralhub/logs/worker-error.log
stdout_logfile=/opt/neuralhub/logs/worker-access.log
environment=PATH="/opt/neuralhub/app/backend/.venv/bin:/opt/neuralhub/.local/bin"
```

**Note**: The worker uses the Python from the virtual environment to ensure consistency.

### 3. Start Services

Update supervisor and start services:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start neuralhub-api
sudo supervisorctl start neuralhub-worker
```

Check status:

```bash
sudo supervisorctl status
```

You should see both services in `RUNNING` state.

---

## Nginx Configuration

### 1. Create Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/neuralhub
```

Add configuration:

```nginx
upstream neuralhub_backend {
    server 127.0.0.1:5003;
}

server {
    listen 80;
    server_name your-domain.com www.your-domain.com;  # Replace with your domain

    client_max_body_size 100M;

    location / {
        proxy_pass http://neuralhub_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support (if needed)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Health check endpoint
    location /health {
        proxy_pass http://neuralhub_backend/health;
        access_log off;
    }
}
```

### 2. Enable Site

```bash
sudo ln -s /etc/nginx/sites-available/neuralhub /etc/nginx/sites-enabled/
sudo nginx -t  # Test configuration
sudo systemctl restart nginx
```

### 3. Setup SSL with Let's Encrypt (Optional but Recommended)

Install Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Get SSL certificate:

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

Follow the prompts. Certbot will automatically configure HTTPS and set up auto-renewal.

---

## CI/CD Pipeline Setup

### 1. Create GitHub Actions Workflow Directory

On your local machine, in the repository root:

```bash
mkdir -p .github/workflows
```

### 2. Create Deployment Workflow

Create file `.github/workflows/deploy-backend.yml`:

```yaml
name: Deploy Backend to Ubuntu Server

on:
  push:
    branches:
      - main
    paths:
      - 'backend/**'
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup SSH Key
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Add Server to Known Hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts

      - name: Deploy Application
        env:
          SERVER_USER: ${{ secrets.SERVER_USER }}
          SERVER_IP: ${{ secrets.SERVER_IP }}
        run: |
          ssh $SERVER_USER@$SERVER_IP << 'ENDSSH'
            set -e
            
            echo "🚀 Starting deployment..."
            
            # Switch to application directory
            cd /opt/neuralhub/app
            
            # Pull latest changes
            echo "📥 Pulling latest code..."
            sudo -u neuralhub git pull origin main
            
            # Update dependencies
            echo "📦 Updating dependencies..."
            cd backend
            sudo -u neuralhub /opt/neuralhub/.cargo/bin/uv sync --frozen
            
            # Run database migrations
            echo "🗄️  Running database migrations..."
            sudo -u neuralhub /opt/neuralhub/.cargo/bin/uv run prisma generate
            sudo -u neuralhub /opt/neuralhub/.cargo/bin/uv run prisma db push
            
            # Restart services
            echo "🔄 Restarting services..."
            sudo supervisorctl restart neuralhub-api
            sudo supervisorctl restart neuralhub-worker
            
            # Wait for services to start
            sleep 5
            
            # Check service status
            echo "✅ Checking service status..."
            sudo supervisorctl status neuralhub-api
            sudo supervisorctl status neuralhub-worker
            
            # Health check
            echo "🏥 Running health check..."
            curl -f http://localhost:5003/health || exit 1
            
            echo "✨ Deployment completed successfully!"
          ENDSSH

      - name: Notify Deployment Success
        if: success()
        run: |
          echo "✅ Backend deployed successfully to production!"

      - name: Notify Deployment Failure
        if: failure()
        run: |
          echo "❌ Backend deployment failed!"
          exit 1
```

### 3. Setup GitHub Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions

Add the following secrets:

1. **SSH_PRIVATE_KEY**: Your local machine's private SSH key that has access to the server
   ```bash
   # On your local machine
   cat ~/.ssh/id_rsa  # or ~/.ssh/id_ed25519
   ```
   Copy the entire content including `-----BEGIN` and `-----END` lines.

2. **SERVER_IP**: Your Ubuntu server's IP address or domain
   ```
   Example: 192.168.1.100 or server.example.com
   ```

3. **SERVER_USER**: Your SSH user (typically your username, not `neuralhub`)
   ```
   Example: ubuntu or your-username
   ```

### 4. Configure SSH Access from GitHub Actions

On your Ubuntu server, ensure your SSH user can sudo without password for the deployment script:

```bash
sudo visudo
```

Add this line (replace `your-username` with your SSH user):

```
your-username ALL=(ALL) NOPASSWD: /usr/bin/supervisorctl, /usr/bin/git, /home/neuralhub/.cargo/bin/uv
```

### 5. Test the Pipeline

Commit and push the workflow file:

```bash
git add .github/workflows/deploy-backend.yml
git commit -m "Add CI/CD deployment pipeline"
git push origin main
```

Go to your GitHub repository → Actions tab to see the workflow running.

---

## Deployment Process

### Automatic Deployment

Every push to the `main` branch that changes files in the `backend/` directory will trigger automatic deployment.

### Manual Deployment

You can also trigger deployment manually:

1. Go to GitHub → Your Repository → Actions
2. Select "Deploy Backend to Ubuntu Server"
3. Click "Run workflow"
4. Select the branch and click "Run workflow"

### Monitoring Deployment

Monitor the deployment in real-time:

```bash
# Watch API logs
sudo tail -f /opt/neuralhub/logs/api-access.log

# Watch worker logs
sudo tail -f /opt/neuralhub/logs/worker-access.log

# Watch error logs
sudo tail -f /opt/neuralhub/logs/api-error.log
```

---

## Monitoring & Maintenance

### 1. Service Management Commands

```bash
# Check service status
sudo supervisorctl status

# Restart API
sudo supervisorctl restart neuralhub-api

# Restart Worker
sudo supervisorctl restart neuralhub-worker

# Stop services
sudo supervisorctl stop neuralhub-api neuralhub-worker

# Start services
sudo supervisorctl start neuralhub-api neuralhub-worker

# View logs
sudo supervisorctl tail -f neuralhub-api
sudo supervisorctl tail -f neuralhub-worker
```

### 2. Health Checks

```bash
# Application health
curl http://localhost:5003/health

# Check if services are listening
sudo netstat -tulpn | grep 5003
```

### 3. Log Rotation

Create log rotation config:

```bash
sudo nano /etc/logrotate.d/neuralhub
```

Add:

```
/opt/neuralhub/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 neuralhub neuralhub
    sharedscripts
    postrotate
        supervisorctl restart neuralhub-api neuralhub-worker > /dev/null 2>&1 || true
    endscript
}
```

### 4. Database Backup

Create backup script:

```bash
sudo nano /opt/neuralhub/backup-db.sh
```

Add:

```bash
#!/bin/bash
BACKUP_DIR="/opt/neuralhub/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# This requires pg_dump with connection to your external database
# Adjust based on your database setup
pg_dump $DATABASE_URL > $BACKUP_DIR/backup_$TIMESTAMP.sql

# Keep only last 7 days of backups
find $BACKUP_DIR -name "backup_*.sql" -mtime +7 -delete

echo "Backup completed: backup_$TIMESTAMP.sql"
```

Make executable:

```bash
sudo chmod +x /opt/neuralhub/backup-db.sh
```

Setup cron job:

```bash
sudo crontab -e
```

Add daily backup at 2 AM:

```
0 2 * * * /opt/neuralhub/backup-db.sh >> /opt/neuralhub/logs/backup.log 2>&1
```

### 5. System Monitoring

Install monitoring tools:

```bash
sudo apt install -y htop iotop nethogs
```

Monitor resources:

```bash
# CPU and Memory
htop

# Disk usage
df -h

# Check API process
ps aux | grep uvicorn

# Check worker process
ps aux | grep worker_runner
```

---

## Troubleshooting

### Cannot Switch to neuralhub User

If you get "This account is currently not available" error:

```bash
# Fix the shell for the neuralhub user
sudo usermod -s /bin/bash neuralhub

# Now you can switch to the user
sudo su - neuralhub
```

Alternatively, you can run commands as the neuralhub user without switching:

```bash
# Run commands as neuralhub user
sudo -u neuralhub bash -c 'cd /opt/neuralhub && git clone ...'
```

### Services Not Starting

Check supervisor logs:

```bash
sudo supervisorctl tail neuralhub-api stderr
sudo supervisorctl tail neuralhub-worker stderr
```

Check if port is already in use:

```bash
sudo lsof -i :5003
```

### Database Connection Issues

Test database connection:

```bash
psql $DATABASE_URL
```

Check if DATABASE_URL is properly set:

```bash
sudo -u neuralhub cat /opt/neuralhub/app/backend/.env | grep DATABASE_URL
```

### Redis Connection Issues

Test Redis connection:

```bash
redis-cli -u $REDIS_URL ping
```

### Deployment Failures

Check GitHub Actions logs for specific errors.

SSH into server and check:

```bash
# Check if code was pulled
cd /opt/neuralhub/app
sudo -u neuralhub git log -1

# Check supervisor status
sudo supervisorctl status

# Check nginx status
sudo systemctl status nginx

# Check nginx logs
sudo tail -f /var/log/nginx/error.log
```

### High Memory Usage

Check which process is using memory:

```bash
sudo ps aux --sort=-%mem | head -10
```

Restart services if needed:

```bash
sudo supervisorctl restart neuralhub-api neuralhub-worker
```

### Disk Space Issues

Check disk usage:

```bash
df -h
du -sh /opt/neuralhub/*
```

Clean up logs:

```bash
sudo find /opt/neuralhub/logs -name "*.log" -mtime +7 -delete
sudo journalctl --vacuum-time=7d
```

---

## Rollback Strategy

### Manual Rollback

If deployment fails, rollback to previous version:

```bash
# SSH into server
ssh username@your-server-ip

# Switch to app directory
cd /opt/neuralhub/app

# View commit history
sudo -u neuralhub git log --oneline -5

# Rollback to specific commit
sudo -u neuralhub git reset --hard COMMIT_HASH

# Update dependencies
cd backend
sudo -u neuralhub /home/neuralhub/.cargo/bin/uv sync --frozen

# Restart services
sudo supervisorctl restart neuralhub-api neuralhub-worker
```

### Create Rollback Workflow

Create `.github/workflows/rollback-backend.yml`:

```yaml
name: Rollback Backend

on:
  workflow_dispatch:
    inputs:
      commit_hash:
        description: 'Commit hash to rollback to'
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

      - name: Rollback Application
        env:
          SERVER_USER: ${{ secrets.SERVER_USER }}
          SERVER_IP: ${{ secrets.SERVER_IP }}
          COMMIT_HASH: ${{ github.event.inputs.commit_hash }}
        run: |
          ssh $SERVER_USER@$SERVER_IP << ENDSSH
            set -e
            
            echo "⏮️  Rolling back to commit $COMMIT_HASH..."
            
            cd /opt/neuralhub/app
            sudo -u neuralhub git reset --hard $COMMIT_HASH
            
            cd backend
            sudo -u neuralhub /opt/neuralhub/.local/bin/uv sync --frozen
            
            sudo supervisorctl restart neuralhub-api neuralhub-worker
            
            sleep 5
            curl -f http://localhost:5003/health || exit 1
            
            echo "✅ Rollback completed successfully!"
          ENDSSH
```

---

## Performance Optimization

### 1. Increase Uvicorn Workers

Edit supervisor config based on your server's CPU cores:

```bash
sudo nano /etc/supervisor/conf.d/neuralhub-api.conf
```

Change `--workers 4` to match your CPU cores (typically CPU cores * 2 + 1):

```ini
command=/opt/neuralhub/app/backend/.venv/bin/uvicorn src.main:app --host 0.0.0.0 --port 5003 --workers 8
```

Restart:

```bash
sudo supervisorctl restart neuralhub-api
```

### 2. Enable Nginx Caching

Edit nginx config:

```bash
sudo nano /etc/nginx/sites-available/neuralhub
```

Add caching:

```nginx
# Add at the top of the file
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m max_size=100m inactive=60m;

server {
    # ... existing config ...
    
    location /api/ {
        proxy_cache api_cache;
        proxy_cache_valid 200 5m;
        proxy_cache_bypass $http_cache_control;
        add_header X-Cache-Status $upstream_cache_status;
        
        # ... existing proxy settings ...
    }
}
```

Restart nginx:

```bash
sudo systemctl restart nginx
```

### 3. Database Connection Pooling

Connection pooling is already configured in SQLAlchemy. Monitor connection usage:

```bash
# Check active connections to database
psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity;"
```

---

## Security Best Practices

### 1. Regular Updates

Setup automatic security updates:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

### 2. Firewall Rules

Ensure only necessary ports are open:

```bash
sudo ufw status
```

### 3. Fail2Ban (Optional)

Install fail2ban to prevent brute force attacks:

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### 4. Environment Variable Security

Never commit `.env` files. Use GitHub Secrets for sensitive data.

Restrict access to .env file:

```bash
sudo chmod 600 /opt/neuralhub/app/backend/.env
sudo chown neuralhub:neuralhub /opt/neuralhub/app/backend/.env
```

### 5. HTTPS Only

Ensure all traffic uses HTTPS. Add redirect in nginx:

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}
```

---

## Scaling Considerations

### Horizontal Scaling

To handle more traffic:

1. **Add Load Balancer**: Use AWS ALB, Nginx, or HAProxy
2. **Multiple Servers**: Deploy to multiple Ubuntu servers
3. **Container Orchestration**: Consider Docker + Kubernetes or Docker Swarm

### Vertical Scaling

Upgrade server resources:

```bash
# Monitor current usage
htop
free -h
df -h
```

Upgrade based on bottlenecks:
- High CPU → More cores
- High Memory → More RAM
- High Disk I/O → SSD or NVMe storage

---

## Additional Resources

### UV Command Reference

Your project uses `uv` (modern Python package manager) instead of traditional pip/poetry:

```bash
# Install dependencies
uv sync --frozen

# Run any command with the virtual environment
uv run <command>

# Examples:
uv run python script.py
uv run uvicorn src.main:app --reload
uv run prisma generate
uv run prisma db push

# Add a new dependency
uv add package-name

# Update dependencies
uv sync
```

**Key Benefits of UV**:
- Automatically manages Python version
- Fast dependency resolution
- No need to manually activate virtual environment
- Similar to npm/yarn for JavaScript

### Useful Commands

```bash
# Check system load
uptime

# Check disk I/O
iostat -x 1

# Check network connections
netstat -an | grep 5003

# Check running processes
ps aux | grep python

# Monitor logs in real-time
tail -f /opt/neuralhub/logs/*.log

# Check supervisor process
ps aux | grep supervisor
```

### Documentation Links

- [FastAPI Production Deployment](https://fastapi.tiangolo.com/deployment/)
- [Uvicorn Deployment](https://www.uvicorn.org/deployment/)
- [Nginx Configuration](https://nginx.org/en/docs/)
- [Supervisor Documentation](http://supervisord.org/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

---

## Summary Checklist

### Initial Setup
- [ ] Ubuntu server configured with SSH access
- [ ] Python 3.11+ and UV installed
- [ ] Nginx and Supervisor installed
- [ ] Application user created
- [ ] Repository cloned
- [ ] Environment variables configured
- [ ] Database migrations completed
- [ ] Services running under Supervisor
- [ ] Nginx reverse proxy configured
- [ ] SSL certificate obtained (optional)

### CI/CD Setup
- [ ] GitHub workflow files created
- [ ] GitHub secrets configured
- [ ] SSH access from GitHub Actions working
- [ ] Deployment pipeline tested
- [ ] Rollback workflow created

### Production Readiness
- [ ] Health checks passing
- [ ] Monitoring setup
- [ ] Log rotation configured
- [ ] Backup strategy implemented
- [ ] Security measures applied
- [ ] Documentation reviewed

---

## Support & Contact

For issues or questions:
1. Check the logs: `/opt/neuralhub/logs/`
2. Review GitHub Actions workflow runs
3. Check service status: `sudo supervisorctl status`
4. Review this documentation

---

**Last Updated**: January 2026  
**Version**: 1.0  
**Maintained by**: NeuralHub Team
